---
layout: post
title: "rtmp chunk封装与解封装"
categories: LiveStream
tags: [documentation, live, rtmp]
description: 关于几种软件对rtmp chunk的封装与解封装
---
# 封装
## FFmpeg实现

{% highlight c %}
int ff_rtmp_packet_write(URLContext *h, RTMPPacket *pkt,
                         int chunk_size, RTMPPacket **prev_pkt_ptr,
                         int *nb_prev_pkt)
{
    uint8_t pkt_hdr[16], *p = pkt_hdr;
    int mode = RTMP_PS_TWELVEBYTES;
    int off = 0;
    int written = 0;
    int ret;
    RTMPPacket *prev_pkt;
    int use_delta; // flag if using timestamp delta, not RTMP_PS_TWELVEBYTES
    uint32_t timestamp; // full 32-bit timestamp or delta value

    if ((ret = ff_rtmp_check_alloc_array(prev_pkt_ptr, nb_prev_pkt,
                                         pkt->channel_id)) < 0)
        return ret;
    prev_pkt = *prev_pkt_ptr;

    /*
    * channel_id就是chunk stream id，extra就是message stream id。当以下一种条件发生时，
    * 发送type0 chunk：1、channel_id为0（不可能发生），2、stream id不同，3、时间戳小于message的时间戳。
    * 否则timestamp为时间戳差值
    **/
    //if channel_id = 0, this is first presentation of prev_pkt, send full hdr.
    use_delta = prev_pkt[pkt->channel_id].channel_id &&
        pkt->extra == prev_pkt[pkt->channel_id].extra &&
        pkt->timestamp >= prev_pkt[pkt->channel_id].timestamp;
    /* 时间戳 */
    timestamp = pkt->timestamp;
    if (use_delta) {
        timestamp -= prev_pkt[pkt->channel_id].timestamp;
    }
    if (timestamp >= 0xFFFFFF) {
        pkt->ts_field = 0xFFFFFF;
    } else {
        pkt->ts_field = timestamp;
    }

    /* 判断chunk type */
    if (use_delta) {
        if (pkt->type == prev_pkt[pkt->channel_id].type &&
            pkt->size == prev_pkt[pkt->channel_id].size) {
            /* type2 */
            mode = RTMP_PS_FOURBYTES;
            if (pkt->ts_field == prev_pkt[pkt->channel_id].ts_field)
                /* type3, 时间戳、大小、message type、stream id与上一个packet相同 */
                mode = RTMP_PS_ONEBYTE;
        } else {
            /* type1 */
            mode = RTMP_PS_EIGHTBYTES;
        }
    }

    if (pkt->channel_id < 64) {
        bytestream_put_byte(&p, pkt->channel_id | (mode << 6));
    } else if (pkt->channel_id < 64 + 256) {
        bytestream_put_byte(&p, 0               | (mode << 6));
        bytestream_put_byte(&p, pkt->channel_id - 64);
    } else {
        bytestream_put_byte(&p, 1               | (mode << 6));
        bytestream_put_le16(&p, pkt->channel_id - 64);
    }
    if (mode != RTMP_PS_ONEBYTE) {
        bytestream_put_be24(&p, pkt->ts_field);
        if (mode != RTMP_PS_FOURBYTES) {
            bytestream_put_be24(&p, pkt->size);
            bytestream_put_byte(&p, pkt->type);
            if (mode == RTMP_PS_TWELVEBYTES)
                bytestream_put_le32(&p, pkt->extra);
        }
    }
    if (pkt->ts_field == 0xFFFFFF)
        bytestream_put_be32(&p, timestamp);
    // save history
    prev_pkt[pkt->channel_id].channel_id = pkt->channel_id;
    prev_pkt[pkt->channel_id].type       = pkt->type;
    prev_pkt[pkt->channel_id].size       = pkt->size;
    prev_pkt[pkt->channel_id].timestamp  = pkt->timestamp;
    prev_pkt[pkt->channel_id].ts_field   = pkt->ts_field;
    prev_pkt[pkt->channel_id].extra      = pkt->extra;

    if ((ret = ffurl_write(h, pkt_hdr, p - pkt_hdr)) < 0)
        return ret;
    written = p - pkt_hdr + pkt->size;
    while (off < pkt->size) {
        int towrite = FFMIN(chunk_size, pkt->size - off);
        if ((ret = ffurl_write(h, pkt->data + off, towrite)) < 0)
            return ret;
        off += towrite;
        if (off < pkt->size) {
            uint8_t marker = 0xC0 | pkt->channel_id;
            if ((ret = ffurl_write(h, &marker, 1)) < 0)
                return ret;
            written++;
            if (pkt->ts_field == 0xFFFFFF) {
                uint8_t ts_header[4];
                AV_WB32(ts_header, timestamp);
                if ((ret = ffurl_write(h, ts_header, 4)) < 0)
                    return ret;
                written += 4;
            }
        }
    }
    return written;
}
{% endhighlight %}

| chunk类型 | 封装条件 |
| -------- | ------- |
| type0 | stream id不同或者时间戳小于上一个message的时间戳 |
| type1 | stream id相同，时间戳大于上一个message时间戳，类型或者大小不同 |
| type2 | stream id相同，时间戳大于上一个message时间戳，类型和大小都相同 |
| type3 | stream id相同，时间戳大于上一个message时间戳，类型和大小都相同，且时间戳变化量与上一个相同或者属于同一个帧 |

| channel_id | 描述 |
| -------- | ----- |
| 2 | RTMP_NETWORK_CHANNEL |
| 3 | RTMP_SYSTEM_CHANNEL |
| 4 | RTMP_AUDIO_CHANNEL |
| 6 | RTMP_VIDEO_CHANNEL |
| 8 | RTMP_SOURCE_CHANNEL |

## SRS实现

{% highlight cpp %}
int nbh = msg->chunk_header(c0c3_cache, nb_cache, p == msg->payload);
{% endhighlight %}

`p`为发送数据的位置，`msg->payload`为消息的起始位置。

{% highlight cpp %}
int SrsSharedPtrMessage::chunk_header(char* cache, int nb_cache, bool c0)
{
    if (c0) {
        return srs_chunk_header_c0(
            ptr->header.perfer_cid, timestamp, ptr->header.payload_length,
            ptr->header.message_type, stream_id,
            cache, nb_cache);
    } else {
        return srs_chunk_header_c3(
            ptr->header.perfer_cid, timestamp,
            cache, nb_cache);
    }
}
{% endhighlight %}

`srs_chunk_header_c0`是封装type0类型的chunk，`srs_chunk_header_c3`是封装type3类型的chunk，所以srs的发送策略为消息的第一个chunk为type0，同一个消息的其余chunk为type3。没有使用type1和type2类型的chunk。

## nginx-rtmp

{% highlight c %}
void
ngx_rtmp_prepare_message(ngx_rtmp_session_t *s, ngx_rtmp_header_t *h,
        ngx_rtmp_header_t *lh, ngx_chain_t *out)
{
    ngx_chain_t                *l;
    u_char                     *p, *pp;
    ngx_int_t                   hsize, thsize, nbufs;
    uint32_t                    mlen, timestamp, ext_timestamp;
    static uint8_t              hdrsize[] = { 12, 8, 4, 1 };
    u_char                      th[7];
    ngx_rtmp_core_main_conf_t  *cmcf;
    ngx_rtmp_core_srv_conf_t   *cscf;
    uint8_t                     fmt;
    ngx_connection_t           *c;

    c = s->connection;

    cmcf = ngx_rtmp_get_module_main_conf(s, ngx_rtmp_core_module);
    cscf = ngx_rtmp_get_module_srv_conf(s, ngx_rtmp_core_module);

    if (h->csid >= (uint32_t)cmcf->max_streams) {
        ngx_log_error(NGX_LOG_INFO, c->log, 0,
                "RTMP out chunk stream too big: %D >= %D",
                h->csid, cmcf->max_streams);
        ngx_rtmp_finalize_session(s);
        return;
    }

    /* detect packet size */
    mlen = 0;
    nbufs = 0;
    for(l = out; l; l = l->next) {
        mlen += (l->buf->last - l->buf->pos);
        ++nbufs;
    }

    fmt = 0;
    if (lh && lh->csid && h->msid == lh->msid) {
        ++fmt;
        if (h->type == lh->type && mlen && mlen == lh->mlen) {
            ++fmt;
            if (h->timestamp == lh->timestamp) {
                ++fmt;
            }
        }
        timestamp = h->timestamp - lh->timestamp;
    } else {
        timestamp = h->timestamp;
    }

    /*if (lh) {
        *lh = *h;
        lh->mlen = mlen;
    }*/

    hsize = hdrsize[fmt];

    ngx_log_debug8(NGX_LOG_DEBUG_RTMP, s->connection->log, 0,
            "RTMP prep %s (%d) fmt=%d csid=%uD timestamp=%uD "
            "mlen=%uD msid=%uD nbufs=%d",
            ngx_rtmp_message_type(h->type), (int)h->type, (int)fmt,
            h->csid, timestamp, mlen, h->msid, nbufs);

    ext_timestamp = 0;
    if (timestamp >= 0x00ffffff) {
        ext_timestamp = timestamp;
        timestamp = 0x00ffffff;
        hsize += 4;
    }

    if (h->csid >= 64) {
        ++hsize;
        if (h->csid >= 320) {
            ++hsize;
        }
    }

    /* fill initial header */
    out->buf->pos -= hsize;
    p = out->buf->pos;

    /* basic header */
    *p = (fmt << 6);
    if (h->csid >= 2 && h->csid <= 63) {
        *p++ |= (((uint8_t)h->csid) & 0x3f);
    } else if (h->csid >= 64 && h->csid < 320) {
        ++p;
        *p++ = (uint8_t)(h->csid - 64);
    } else {
        *p++ |= 1;
        *p++ = (uint8_t)(h->csid - 64);
        *p++ = (uint8_t)((h->csid - 64) >> 8);
    }

    /* create fmt3 header for successive fragments */
    thsize = p - out->buf->pos;
    ngx_memcpy(th, out->buf->pos, thsize);
    th[0] |= 0xc0;

    /* message header */
    if (fmt <= 2) {
        pp = (u_char*)&timestamp;
        *p++ = pp[2];
        *p++ = pp[1];
        *p++ = pp[0];
        if (fmt <= 1) {
            pp = (u_char*)&mlen;
            *p++ = pp[2];
            *p++ = pp[1];
            *p++ = pp[0];
            *p++ = h->type;
            if (fmt == 0) {
                pp = (u_char*)&h->msid;
                *p++ = pp[0];
                *p++ = pp[1];
                *p++ = pp[2];
                *p++ = pp[3];
            }
        }
    }

    /* extended header */
    if (ext_timestamp) {
        pp = (u_char*)&ext_timestamp;
        *p++ = pp[3];
        *p++ = pp[2];
        *p++ = pp[1];
        *p++ = pp[0];

        /* This CONTRADICTS the standard
         * but that's the way flash client
         * wants data to be encoded;
         * ffmpeg complains */
        if (cscf->play_time_fix) {
            ngx_memcpy(&th[thsize], p - 4, 4);
            thsize += 4;
        }
    }

    /* append headers to successive fragments */
    for(out = out->next; out; out = out->next) {
        out->buf->pos -= thsize;
        ngx_memcpy(out->buf->pos, th, thsize);
    }
}
{% endhighlight %}

| chunk类型 | 封装条件 |
| -------- | ------- |
| type0 | 第一个chunk或者message stream id与前一个message不同 |
| type1 | stream id相同，类型或者大小不同 |
| type2 | stream id相同，类型和大小都相同，时间戳不同 |
| type3 | stream id相同，类型和大小都相同，且时间戳变化量与上一个相同或者属于同一个帧 |

| channel_id | 描述 |
| --------- | ---- |
| 3 | NGX_RTMP_CSID_AMF_INI |
| 5 | NGX_RTMP_CSID_AMF |
| 6 | NGX_RTMP_CSID_AUDIO |
| 7 | NGX_RTMP_CSID_VIDEO |

## sanma
int RTMPEncoder::messageToChunks(RTMPChunk** chunks, const RTMPMessage* msg, unsigned int chunk_stream_id)

{% highlight c %}
if ( m_last_chunk.m_basic_header.m_chunk_stream_id != chunk_stream_id ||
        force_new_chunk == true ||
        m_last_chunk.m_msg_header.m_message_stream_id != msg->m_stream_id ) {
    m_abs_timestamp = 1;
    force_new_chunk = false;
}

// force new chunk when msg_length > 2 x chunk_size
if (chunk_num >= 2 && m_abs_timestamp == 0 && i == 0)
{
    m_abs_timestamp = 1;
    force_new_chunk = false;
}

if (0 == m_abs_timestamp) {
    if (chunk->m_msg_header.m_timestamp < m_channel_timestamp[chunk->m_basic_header.m_chunk_stream_id])
    {
        chunk->m_msg_header.m_timestamp = m_channel_timestamp[chunk->m_basic_header.m_chunk_stream_id];
    }

    chunk->m_msg_header.m_timestamp -= m_channel_timestamp[chunk->m_basic_header.m_chunk_stream_id];
    chunk->m_msg_header.m_timestamp -= timestamp_gep;
    if (chunk->m_msg_header.m_timestamp > 0x7fffffff) {
        chunk->m_msg_header.m_timestamp = 0;
    }
}

if (1 == m_abs_timestamp) {
    chunk->m_basic_header.m_chunk_type = CHUNK_TYPE_NEW;
} else {
    if (m_last_chunk.m_msg_header.m_message_length != msg->m_length
        || m_last_chunk.m_msg_header.m_message_type !=  msg->m_type
        || ((m_last_chunk.m_basic_header.m_chunk_type == CHUNK_TYPE_NEW) && (msg->m_length <= m_max_chunk_size)))
    {
        chunk->m_basic_header.m_chunk_type = CHUNK_TYPE_SAME_STREAM;
    }
    else if(0 != chunk->m_msg_header.m_timestamp)
    {
        chunk->m_basic_header.m_chunk_type = CHUNK_TYPE_TIME_CHANGE;
    }
    else
    {
        chunk->m_basic_header.m_chunk_type = CHUNK_TYPE_SAME_MESSAGE;
    }
}
{% endhighlight %}

| chunk类型 | 封装条件 |
| -------- | ------- |
| type0 | chunk stream id、message stream id与前一个message不同或者当一个message拆分成多个message后的第一个chunk |
| type1 | stream id相同，类型或者大小不同，或者上一个chunk为type0且message长度小于chunk size |
| type2 | stream id相同，类型和大小都相同，时间戳不同 |
| type3 | stream id相同，类型和大小都相同，且时间戳变化量与上一个相同或者属于同一个帧 |

## spider
同nginx rtmp

# 解封装
chunk basic header的解封装都大同小异，主要看message header部分的解封装
## SRS

{% highlight cpp %}
/**
* parse the message header.
*   3bytes: timestamp delta,    fmt=0,1,2
*   3bytes: payload length,     fmt=0,1
*   1bytes: message type,       fmt=0,1
*   4bytes: stream id,          fmt=0
* where:
*   fmt=0, 0x0X
*   fmt=1, 0x4X
*   fmt=2, 0x8X
*   fmt=3, 0xCX
*/
int SrsProtocol::read_message_header(SrsChunkStream* chunk, char fmt)
{
    int ret = ERROR_SUCCESS;
    
    /**
    * we should not assert anything about fmt, for the first packet.
    * (when first packet, the chunk->msg is NULL).
    * the fmt maybe 0/1/2/3, the FMLE will send a 0xC4 for some audio packet.
    * the previous packet is:
    *     04                // fmt=0, cid=4
    *     00 00 1a          // timestamp=26
    *     00 00 9d          // payload_length=157
    *     08                // message_type=8(audio)
    *     01 00 00 00       // stream_id=1
    * the current packet maybe:
    *     c4             // fmt=3, cid=4
    * it's ok, for the packet is audio, and timestamp delta is 26.
    * the current packet must be parsed as:
    *     fmt=0, cid=4
    *     timestamp=26+26=52
    *     payload_length=157
    *     message_type=8(audio)
    *     stream_id=1
    * so we must update the timestamp even fmt=3 for first packet.
    */
    // fresh packet used to update the timestamp even fmt=3 for first packet.
    // fresh packet always means the chunk is the first one of message.
    bool is_first_chunk_of_msg = !chunk->msg;
    
    // but, we can ensure that when a chunk stream is fresh, 
    // the fmt must be 0, a new stream.
    if (chunk->msg_count == 0 && fmt != RTMP_FMT_TYPE0) {
        // for librtmp, if ping, it will send a fresh stream with fmt=1,
        // 0x42             where: fmt=1, cid=2, protocol contorl user-control message
        // 0x00 0x00 0x00   where: timestamp=0
        // 0x00 0x00 0x06   where: payload_length=6
        // 0x04             where: message_type=4(protocol control user-control message)
        // 0x00 0x06            where: event Ping(0x06)
        // 0x00 0x00 0x0d 0x0f  where: event data 4bytes ping timestamp.
        // @see: https://github.com/ossrs/srs/issues/98
        if (chunk->cid == RTMP_CID_ProtocolControl && fmt == RTMP_FMT_TYPE1) {
            srs_warn("accept cid=2, fmt=1 to make librtmp happy.");
        } else {
            // must be a RTMP protocol level error.
            ret = ERROR_RTMP_CHUNK_START;
            srs_error("chunk stream is fresh, fmt must be %d, actual is %d. cid=%d, ret=%d", 
                RTMP_FMT_TYPE0, fmt, chunk->cid, ret);
            return ret;
        }
    }

    // when exists cache msg, means got an partial message,
    // the fmt must not be type0 which means new message.
    if (chunk->msg && fmt == RTMP_FMT_TYPE0) {
        ret = ERROR_RTMP_CHUNK_START;
        srs_error("chunk stream exists, "
            "fmt must not be %d, actual is %d. ret=%d", RTMP_FMT_TYPE0, fmt, ret);
        return ret;
    }
    
    // create msg when new chunk stream start
    if (!chunk->msg) {
        chunk->msg = new SrsCommonMessage();
        srs_verbose("create message for new chunk, fmt=%d, cid=%d", fmt, chunk->cid);
    }

    // read message header from socket to buffer.
    static char mh_sizes[] = {11, 7, 3, 0};
    int mh_size = mh_sizes[(int)fmt];
    srs_verbose("calc chunk message header size. fmt=%d, mh_size=%d", fmt, mh_size);
    
    if (mh_size > 0 && (ret = in_buffer->grow(skt, mh_size)) != ERROR_SUCCESS) {
        if (ret != ERROR_SOCKET_TIMEOUT && !srs_is_client_gracefully_close(ret)) {
            srs_error("read %dbytes message header failed. ret=%d", mh_size, ret);
        }
        return ret;
    }
    
    /**
    * parse the message header.
    *   3bytes: timestamp delta,    fmt=0,1,2
    *   3bytes: payload length,     fmt=0,1
    *   1bytes: message type,       fmt=0,1
    *   4bytes: stream id,          fmt=0
    * where:
    *   fmt=0, 0x0X
    *   fmt=1, 0x4X
    *   fmt=2, 0x8X
    *   fmt=3, 0xCX
    */
    // see also: ngx_rtmp_recv
    if (fmt <= RTMP_FMT_TYPE2) {
        char* p = in_buffer->read_slice(mh_size);
    
        char* pp = (char*)&chunk->header.timestamp_delta;
        pp[2] = *p++;
        pp[1] = *p++;
        pp[0] = *p++;
        pp[3] = 0;
        
        // fmt: 0
        // timestamp: 3 bytes
        // If the timestamp is greater than or equal to 16777215
        // (hexadecimal 0x00ffffff), this value MUST be 16777215, and the
        // 'extended timestamp header' MUST be present. Otherwise, this value
        // SHOULD be the entire timestamp.
        //
        // fmt: 1 or 2
        // timestamp delta: 3 bytes
        // If the delta is greater than or equal to 16777215 (hexadecimal
        // 0x00ffffff), this value MUST be 16777215, and the 'extended
        // timestamp header' MUST be present. Otherwise, this value SHOULD be
        // the entire delta.
        chunk->extended_timestamp = (chunk->header.timestamp_delta >= RTMP_EXTENDED_TIMESTAMP);
        if (!chunk->extended_timestamp) {
            // Extended timestamp: 0 or 4 bytes
            // This field MUST be sent when the normal timsestamp is set to
            // 0xffffff, it MUST NOT be sent if the normal timestamp is set to
            // anything else. So for values less than 0xffffff the normal
            // timestamp field SHOULD be used in which case the extended timestamp
            // MUST NOT be present. For values greater than or equal to 0xffffff
            // the normal timestamp field MUST NOT be used and MUST be set to
            // 0xffffff and the extended timestamp MUST be sent.
            if (fmt == RTMP_FMT_TYPE0) {
                // 6.1.2.1. Type 0
                // For a type-0 chunk, the absolute timestamp of the message is sent
                // here.
                chunk->header.timestamp = chunk->header.timestamp_delta;
            } else {
                // 6.1.2.2. Type 1
                // 6.1.2.3. Type 2
                // For a type-1 or type-2 chunk, the difference between the previous
                // chunk's timestamp and the current chunk's timestamp is sent here.
                chunk->header.timestamp += chunk->header.timestamp_delta;
            }
        }
        
        if (fmt <= RTMP_FMT_TYPE1) {
            int32_t payload_length = 0;
            pp = (char*)&payload_length;
            pp[2] = *p++;
            pp[1] = *p++;
            pp[0] = *p++;
            pp[3] = 0;
            
            // for a message, if msg exists in cache, the size must not changed.
            // always use the actual msg size to compare, for the cache payload length can changed,
            // for the fmt type1(stream_id not changed), user can change the payload 
            // length(it's not allowed in the continue chunks).
            if (!is_first_chunk_of_msg && chunk->header.payload_length != payload_length) {
                ret = ERROR_RTMP_PACKET_SIZE;
                srs_error("msg exists in chunk cache, "
                    "size=%d cannot change to %d, ret=%d", 
                    chunk->header.payload_length, payload_length, ret);
                return ret;
            }
            
            chunk->header.payload_length = payload_length;
            chunk->header.message_type = *p++;
            
            if (fmt == RTMP_FMT_TYPE0) {
                pp = (char*)&chunk->header.stream_id;
                pp[0] = *p++;
                pp[1] = *p++;
                pp[2] = *p++;
                pp[3] = *p++;
                srs_verbose("header read completed. fmt=%d, mh_size=%d, ext_time=%d, time=%"PRId64", payload=%d, type=%d, sid=%d", 
                    fmt, mh_size, chunk->extended_timestamp, chunk->header.timestamp, chunk->header.payload_length, 
                    chunk->header.message_type, chunk->header.stream_id);
            } else {
                srs_verbose("header read completed. fmt=%d, mh_size=%d, ext_time=%d, time=%"PRId64", payload=%d, type=%d", 
                    fmt, mh_size, chunk->extended_timestamp, chunk->header.timestamp, chunk->header.payload_length, 
                    chunk->header.message_type);
            }
        } else {
            srs_verbose("header read completed. fmt=%d, mh_size=%d, ext_time=%d, time=%"PRId64"", 
                fmt, mh_size, chunk->extended_timestamp, chunk->header.timestamp);
        }
    } else {
        // update the timestamp even fmt=3 for first chunk packet
        if (is_first_chunk_of_msg && !chunk->extended_timestamp) {
            chunk->header.timestamp += chunk->header.timestamp_delta;
        }
        srs_verbose("header read completed. fmt=%d, size=%d, ext_time=%d", 
            fmt, mh_size, chunk->extended_timestamp);
    }
    
    // read extended-timestamp
    if (chunk->extended_timestamp) {
        mh_size += 4;
        srs_verbose("read header ext time. fmt=%d, ext_time=%d, mh_size=%d", fmt, chunk->extended_timestamp, mh_size);
        if ((ret = in_buffer->grow(skt, 4)) != ERROR_SUCCESS) {
            if (ret != ERROR_SOCKET_TIMEOUT && !srs_is_client_gracefully_close(ret)) {
                srs_error("read %dbytes message header failed. required_size=%d, ret=%d", mh_size, 4, ret);
            }
            return ret;
        }
        // the ptr to the slice maybe invalid when grow()
        // reset the p to get 4bytes slice.
        char* p = in_buffer->read_slice(4);

        u_int32_t timestamp = 0x00;
        char* pp = (char*)&timestamp;
        pp[3] = *p++;
        pp[2] = *p++;
        pp[1] = *p++;
        pp[0] = *p++;

        // always use 31bits timestamp, for some server may use 32bits extended timestamp.
        // @see https://github.com/ossrs/srs/issues/111
        timestamp &= 0x7fffffff;
        
        /**
        * RTMP specification and ffmpeg/librtmp is false,
        * but, adobe changed the specification, so flash/FMLE/FMS always true.
        * default to true to support flash/FMLE/FMS.
        * 
        * ffmpeg/librtmp may donot send this filed, need to detect the value.
        * @see also: http://blog.csdn.net/win_lin/article/details/13363699
        * compare to the chunk timestamp, which is set by chunk message header
        * type 0,1 or 2.
        *
        * @remark, nginx send the extended-timestamp in sequence-header,
        * and timestamp delta in continue C1 chunks, and so compatible with ffmpeg,
        * that is, there is no continue chunks and extended-timestamp in nginx-rtmp.
        *
        * @remark, srs always send the extended-timestamp, to keep simple,
        * and compatible with adobe products.
        */
        u_int32_t chunk_timestamp = (u_int32_t)chunk->header.timestamp;
        
        /**
        * if chunk_timestamp<=0, the chunk previous packet has no extended-timestamp,
        * always use the extended timestamp.
        */
        /**
        * about the is_first_chunk_of_msg.
        * @remark, for the first chunk of message, always use the extended timestamp.
        */
        if (!is_first_chunk_of_msg && chunk_timestamp > 0 && chunk_timestamp != timestamp) {
            mh_size -= 4;
            in_buffer->skip(-4);
            srs_info("no 4bytes extended timestamp in the continued chunk");
        } else {
            chunk->header.timestamp = timestamp;
        }
        srs_verbose("header read ext_time completed. time=%"PRId64"", chunk->header.timestamp);
    }
    
    // the extended-timestamp must be unsigned-int,
    //         24bits timestamp: 0xffffff = 16777215ms = 16777.215s = 4.66h
    //         32bits timestamp: 0xffffffff = 4294967295ms = 4294967.295s = 1193.046h = 49.71d
    // because the rtmp protocol says the 32bits timestamp is about "50 days":
    //         3. Byte Order, Alignment, and Time Format
    //                Because timestamps are generally only 32 bits long, they will roll
    //                over after fewer than 50 days.
    // 
    // but, its sample says the timestamp is 31bits:
    //         An application could assume, for example, that all 
    //        adjacent timestamps are within 2^31 milliseconds of each other, so
    //        10000 comes after 4000000000, while 3000000000 comes before
    //        4000000000.
    // and flv specification says timestamp is 31bits:
    //        Extension of the Timestamp field to form a SI32 value. This
    //        field represents the upper 8 bits, while the previous
    //        Timestamp field represents the lower 24 bits of the time in
    //        milliseconds.
    // in a word, 31bits timestamp is ok.
    // convert extended timestamp to 31bits.
    chunk->header.timestamp &= 0x7fffffff;
    
    // valid message, the payload_length is 24bits,
    // so it should never be negative.
    srs_assert(chunk->header.payload_length >= 0);
    
    // copy header to msg
    chunk->msg->header = chunk->header;
    
    // increase the msg count, the chunk stream can accept fmt=1/2/3 message now.
    chunk->msg_count++;
    
    return ret;
}
{% endhighlight %}

## nginx rtmp

{% highlight c %}
static void
ngx_rtmp_recv(ngx_event_t *rev)
{
    ngx_int_t                   n;
    ngx_connection_t           *c;
    ngx_rtmp_session_t         *s;
    ngx_rtmp_core_srv_conf_t   *cscf;
    ngx_rtmp_header_t          *h;
    ngx_rtmp_stream_t          *st, *st0;
    ngx_chain_t                *in, *head;
    ngx_buf_t                  *b;
    u_char                     *p, *pp, *old_pos;
    size_t                      size, fsize, old_size;
    uint8_t                     fmt, ext;
    uint32_t                    csid, timestamp;

    c = rev->data;
    s = c->data;
    b = NULL;
    old_pos = NULL;
    old_size = 0;
    cscf = ngx_rtmp_get_module_srv_conf(s, ngx_rtmp_core_module);

    if (c->destroyed) {
        return;
    }

    for( ;; ) {

        st = &s->in_streams[s->in_csid];

        /* allocate new buffer */
        if (st->in == NULL) {
            st->in = ngx_rtmp_alloc_in_buf(s);
            if (st->in == NULL) {
                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                        "in buf alloc failed");
                ngx_rtmp_finalize_session(s);
                return;
            }
        }

        h  = &st->hdr;
        in = st->in;
        b  = in->buf;

        if (old_size) {

            ngx_log_debug1(NGX_LOG_DEBUG_RTMP, c->log, 0,
                    "reusing formerly read data: %d", old_size);

            b->pos = b->start;
            b->last = ngx_movemem(b->pos, old_pos, old_size);

            if (s->in_chunk_size_changing) {
                ngx_rtmp_finalize_set_chunk_size(s);
            }

        } else {

            if (old_pos) {
                b->pos = b->last = b->start;
            }

            n = c->recv(c, b->last, b->end - b->last);

            if (n == NGX_ERROR || n == 0) {
                ngx_rtmp_finalize_session(s);
                return;
            }

            if (n == NGX_AGAIN) {
                if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
                    ngx_rtmp_finalize_session(s);
                }
                return;
            }

            s->ping_reset = 1;
            ngx_rtmp_update_bandwidth(&ngx_rtmp_bw_in, n);
            b->last += n;
            s->in_bytes += n;

            if (s->in_bytes >= 0xf0000000) {
                ngx_log_debug0(NGX_LOG_DEBUG_RTMP, c->log, 0,
                               "resetting byte counter");
                s->in_bytes = 0;
                s->in_last_ack = 0;
            }

            if (s->ack_size && s->in_bytes - s->in_last_ack >= s->ack_size) {

                s->in_last_ack = s->in_bytes;

                ngx_log_debug1(NGX_LOG_DEBUG_RTMP, c->log, 0,
                        "sending RTMP ACK(%uD)", s->in_bytes);

                if (ngx_rtmp_send_ack(s, s->in_bytes)) {
                    ngx_rtmp_finalize_session(s);
                    return;
                }
            }
        }

        old_pos = NULL;
        old_size = 0;

        /* parse headers */
        if (b->pos == b->start) {
            p = b->pos;

            /* chunk basic header */
            fmt  = (*p >> 6) & 0x03;
            csid = *p++ & 0x3f;

            if (csid == 0) {
                if (b->last - p < 1)
                    continue;
                csid = 64;
                csid += *(uint8_t*)p++;

            } else if (csid == 1) {
                if (b->last - p < 2)
                    continue;
                csid = 64;
                csid += *(uint8_t*)p++;
                csid += (uint32_t)256 * (*(uint8_t*)p++);
            }

            ngx_log_debug2(NGX_LOG_DEBUG_RTMP, c->log, 0,
                    "RTMP bheader fmt=%d csid=%D",
                    (int)fmt, csid);

            if (csid >= (uint32_t)cscf->max_streams) {
                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                    "RTMP in chunk stream too big: %D >= %D",
                    csid, cscf->max_streams);
                ngx_rtmp_finalize_session(s);
                return;
            }

            /* link orphan */
            if (s->in_csid == 0) {

                /* unlink from stream #0 */
                st->in = st->in->next;

                /* link to new stream */
                s->in_csid = csid;
                st = &s->in_streams[csid];
                if (st->in == NULL) {
                    in->next = in;
                } else {
                    in->next = st->in->next;
                    st->in->next = in;
                }
                st->in = in;
                h = &st->hdr;
                h->csid = csid;
            }

            ext = st->ext;
            timestamp = st->dtime;
            if (fmt <= 2 ) {
                if (b->last - p < 3)
                    continue;
                /* timestamp:
                 *  big-endian 3b -> little-endian 4b */
                pp = (u_char*)&timestamp;
                pp[2] = *p++;
                pp[1] = *p++;
                pp[0] = *p++;
                pp[3] = 0;

                ext = (timestamp == 0x00ffffff);

                if (fmt <= 1) {
                    if (b->last - p < 4)
                        continue;
                    /* size:
                     *  big-endian 3b -> little-endian 4b
                     * type:
                     *  1b -> 1b*/
                    pp = (u_char*)&h->mlen;
                    pp[2] = *p++;
                    pp[1] = *p++;
                    pp[0] = *p++;
                    pp[3] = 0;
                    h->type = *(uint8_t*)p++;

                    if (fmt == 0) {
                        if (b->last - p < 4)
                            continue;
                        /* stream:
                         *  little-endian 4b -> little-endian 4b */
                        pp = (u_char*)&h->msid;
                        pp[0] = *p++;
                        pp[1] = *p++;
                        pp[2] = *p++;
                        pp[3] = *p++;
                    }
                }
            }

            /* extended header */
            if (ext) {
                if (b->last - p < 4)
                    continue;
                pp = (u_char*)&timestamp;
                pp[3] = *p++;
                pp[2] = *p++;
                pp[1] = *p++;
                pp[0] = *p++;
            }

            if (st->len == 0) {
                /* Messages with type=3 should
                 * never have ext timestamp field
                 * according to standard.
                 * However that's not always the case
                 * in real life */
                st->ext = (ext && cscf->publish_time_fix);
                if (fmt) {
                    st->dtime = timestamp;
                } else {
                    h->timestamp = timestamp;
                    st->dtime = 0;
                }
            }

            ngx_log_debug8(NGX_LOG_DEBUG_RTMP, c->log, 0,
                    "RTMP mheader fmt=%d %s (%d) "
                    "time=%uD+%uD mlen=%D len=%D msid=%D",
                    (int)fmt, ngx_rtmp_message_type(h->type), (int)h->type,
                    h->timestamp, st->dtime, h->mlen, st->len, h->msid);

            /* header done */
            b->pos = p;

            if (h->mlen > cscf->max_message) {
                ngx_log_error(NGX_LOG_INFO, c->log, 0,
                        "too big message: %uz", cscf->max_message);
                ngx_rtmp_finalize_session(s);
                return;
            }
        }

        size = b->last - b->pos;
        fsize = h->mlen - st->len;

        if (size < ngx_min(fsize, s->in_chunk_size))
            continue;

        /* buffer is ready */

        if (fsize > s->in_chunk_size) {
            /* collect fragmented chunks */
            st->len += s->in_chunk_size;
            b->last = b->pos + s->in_chunk_size;
            old_pos = b->last;
            old_size = size - s->in_chunk_size;

        } else {
            /* handle! */
            head = st->in->next;
            st->in->next = NULL;
            b->last = b->pos + fsize;
            old_pos = b->last;
            old_size = size - fsize;
            st->len = 0;
            h->timestamp += st->dtime;

            if (ngx_rtmp_receive_message(s, h, head) != NGX_OK) {
                ngx_rtmp_finalize_session(s);
                return;
            }

            if (s->in_chunk_size_changing) {
                /* copy old data to a new buffer */
                if (!old_size) {
                    ngx_rtmp_finalize_set_chunk_size(s);
                }

            } else {
                /* add used bufs to stream #0 */
                st0 = &s->in_streams[0];
                st->in->next = st0->in;
                st0->in = head;
                st->in = NULL;
            }
        }

        s->in_csid = 0;
    }
}
{% endhighlight %}

## sanma

{% highlight c %}
int RTMPDecoder::decodeChunkMsgHeader(RTMPChunkMsgHeader* msg_hdr, const char* data, int size,
				const RTMPChunkBasicHeader* basic_hdr, int * decoded_len)
{
	RTMPChunkStream* chunk_stream = getChunkStream(basic_hdr->m_chunk_stream_id);
	if (chunk_stream == NULL) {
		xlwarn(g_run_log, "rtmp:chunk decoder, invalid chunk stream (id=%d)",
		       basic_hdr->m_chunk_stream_id);
		*decoded_len = 0;
		return DECODE_ERROR;
	}

	RTMPChunk *last_chunk = chunk_stream->m_last_chunk;

	static const int msg_header_size_map[4] = {11, 7, 3, 0};
	int msg_header_size = msg_header_size_map[basic_hdr->m_chunk_type];
	if (msg_header_size > size) {
		*decoded_len = 0;
		return DECODE_NEED_MORE_DATA;
	}

	if ( !last_chunk && basic_hdr->m_chunk_type != CHUNK_TYPE_NEW ) {
		xlerror(g_run_log, "rtmp:missing_last_chunk "
			"chunk type:%u id:%u data-size:%u",
			(unsigned int)basic_hdr->m_chunk_type,
			basic_hdr->m_chunk_stream_id, size);

		//SDebug::instance()->PrintHexData(data, 100);
		*decoded_len = 0;
		return DECODE_ERROR;
	}

	if ( last_chunk ) {
		*msg_hdr = last_chunk->m_msg_header;
	}

	const unsigned char* msg_header_buffer = (unsigned char*)data;

	switch (basic_hdr->m_chunk_type) {
	case CHUNK_TYPE_NEW:
		m_abs_timestamp = 1;
		msg_hdr->m_timestamp = GetInt24FromBuffer_BigEndian(msg_header_buffer + 0);
		msg_hdr->m_message_length = GetInt24FromBuffer_BigEndian(msg_header_buffer + 3);
		msg_hdr->m_message_type = msg_header_buffer[6];
		msg_hdr->m_message_stream_id = GetInt32FromBuffer(msg_header_buffer + 7);
		break;

	case CHUNK_TYPE_SAME_STREAM:
		m_abs_timestamp = 0;
		msg_hdr->m_timestamp = GetInt24FromBuffer_BigEndian(msg_header_buffer + 0);
		msg_hdr->m_message_length = GetInt24FromBuffer_BigEndian(msg_header_buffer + 3);
		msg_hdr->m_message_type = msg_header_buffer[6];
		break;

	case CHUNK_TYPE_TIME_CHANGE:
		m_abs_timestamp = 0;
		msg_hdr->m_timestamp = GetInt24FromBuffer_BigEndian(msg_header_buffer + 0);
		break;

	case CHUNK_TYPE_SAME_MESSAGE:
		m_abs_timestamp = 0;
		break;

	default:
		break;
	}

	if (msg_hdr->m_message_length == 0) {
		GlobalDebug("rtmp:chunk message_length:0 header:%d", msg_header_size);
		//SDebug::instance()->PrintHexData(data, msg_header_size);
	}

	*decoded_len = msg_header_size;
	return DECODE_OK;
}
{% endhighlight %}

## FFmpeg

{% highlight c %}
static int rtmp_packet_read_one_chunk(URLContext *h, RTMPPacket *p,
                                      int chunk_size, RTMPPacket **prev_pkt_ptr,
                                      int *nb_prev_pkt, uint8_t hdr)
{

    uint8_t buf[16];
    int channel_id, timestamp, size;
    uint32_t ts_field; // non-extended timestamp or delta field
    uint32_t extra = 0;
    enum RTMPPacketType type;
    int written = 0;
    int ret, toread;
    RTMPPacket *prev_pkt;

    written++;
    channel_id = hdr & 0x3F;

    if (channel_id < 2) { //special case for channel number >= 64
        buf[1] = 0;
        if (ffurl_read_complete(h, buf, channel_id + 1) != channel_id + 1)
            return AVERROR(EIO);
        written += channel_id + 1;
        channel_id = AV_RL16(buf) + 64;
    }
    if ((ret = ff_rtmp_check_alloc_array(prev_pkt_ptr, nb_prev_pkt,
                                         channel_id)) < 0)
        return ret;
    prev_pkt = *prev_pkt_ptr;
    size  = prev_pkt[channel_id].size;
    type  = prev_pkt[channel_id].type;
    extra = prev_pkt[channel_id].extra;

    hdr >>= 6; // header size indicator
    if (hdr == RTMP_PS_ONEBYTE) {
        ts_field = prev_pkt[channel_id].ts_field;
    } else {
        if (ffurl_read_complete(h, buf, 3) != 3)
            return AVERROR(EIO);
        written += 3;
        ts_field = AV_RB24(buf);
        if (hdr != RTMP_PS_FOURBYTES) {
            if (ffurl_read_complete(h, buf, 3) != 3)
                return AVERROR(EIO);
            written += 3;
            size = AV_RB24(buf);
            if (ffurl_read_complete(h, buf, 1) != 1)
                return AVERROR(EIO);
            written++;
            type = buf[0];
            if (hdr == RTMP_PS_TWELVEBYTES) {
                if (ffurl_read_complete(h, buf, 4) != 4)
                    return AVERROR(EIO);
                written += 4;
                extra = AV_RL32(buf);
            }
        }
    }
    if (ts_field == 0xFFFFFF) {
        if (ffurl_read_complete(h, buf, 4) != 4)
            return AVERROR(EIO);
        timestamp = AV_RB32(buf);
    } else {
        timestamp = ts_field;
    }
    if (hdr != RTMP_PS_TWELVEBYTES)
        timestamp += prev_pkt[channel_id].timestamp;

    if (prev_pkt[channel_id].read && size != prev_pkt[channel_id].size) {
        av_log(h, AV_LOG_ERROR, "RTMP packet size mismatch %d != %d\n",
                                size, prev_pkt[channel_id].size);
        ff_rtmp_packet_destroy(&prev_pkt[channel_id]);
        prev_pkt[channel_id].read = 0;
        return AVERROR_INVALIDDATA;
    }

    if (!prev_pkt[channel_id].read) {
        if ((ret = ff_rtmp_packet_create(p, channel_id, type, timestamp,
                                         size)) < 0)
            return ret;
        p->read = written;
        p->offset = 0;
        prev_pkt[channel_id].ts_field   = ts_field;
        prev_pkt[channel_id].timestamp  = timestamp;
    } else {
        // previous packet in this channel hasn't completed reading
        RTMPPacket *prev = &prev_pkt[channel_id];
        p->data          = prev->data;
        p->size          = prev->size;
        p->channel_id    = prev->channel_id;
        p->type          = prev->type;
        p->ts_field      = prev->ts_field;
        p->extra         = prev->extra;
        p->offset        = prev->offset;
        p->read          = prev->read + written;
        p->timestamp     = prev->timestamp;
        prev->data       = NULL;
    }
    p->extra = extra;
    // save history
    prev_pkt[channel_id].channel_id = channel_id;
    prev_pkt[channel_id].type       = type;
    prev_pkt[channel_id].size       = size;
    prev_pkt[channel_id].extra      = extra;
    size = size - p->offset;

    toread = FFMIN(size, chunk_size);
    if (ffurl_read_complete(h, p->data + p->offset, toread) != toread) {
        ff_rtmp_packet_destroy(p);
        return AVERROR(EIO);
    }
    size      -= toread;
    p->read   += toread;
    p->offset += toread;

    if (size > 0) {
       RTMPPacket *prev = &prev_pkt[channel_id];
       prev->data = p->data;
       prev->read = p->read;
       prev->offset = p->offset;
       p->data      = NULL;
       return AVERROR(EAGAIN);
    }

    prev_pkt[channel_id].read = 0; // read complete; reset if needed
    return p->read;
}
{% endhighlight %}