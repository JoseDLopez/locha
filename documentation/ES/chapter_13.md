# 13. Procesamiento de mensajes de ruta RREQ.

El procesamiento de mensajes tiene este esquema general:
- Recibir mensajes entrantes de ruta.
- Actualizar tablas de rutas según corresponda.
- Responder según sea necesario, a menudo regenerando el mensaje entrante con información actualizada.

Después de procesar un mensaje, la información se almacena en la tabla de rutas. Por este motivo es apropiado establecer valores en los campos de mensajes salientes, utilizando la información de la tabla de rutas o los campos del mensaje entrante.

Para recibir un mensaje de algún tipo en un nodo, se debe esperar a la entrada de mensajes UDP, el cual se lleva  acabo en la funcion que representa el thread que escucha los mensajes UDP ademas de los mensajes internos de la aplicación llamados IPC, de los que se hablo en secciones anteriores.

Veamos la seccion de código donde se reciben los mensajes UDP provenientes de nodos remotos.

```cpp
static void *_event_loop(void *arg)
{
    (void)arg;
    msg_t msg;
    msg_t reply;
    msg_t msg_queue[CONFIG_AODVV2_RFC5444_MSG_QUEUE_SIZE];

    /* Initialize message queue */
    msg_init_queue(msg_queue, CONFIG_AODVV2_RFC5444_MSG_QUEUE_SIZE);

    reply.content.value = (uint32_t)(-ENOTSUP);
    reply.type = GNRC_NETAPI_MSG_TYPE_ACK;

    while (1) {
        msg_receive(&msg);

        switch (msg.type) {
            case AODVV2_MSG_TYPE_SEND_RREQ:
                DEBUG("AODVV2_MSG_TYPE_SEND_RREQ\n");
                {
                    aodvv2_msg_t m;
                    memcpy(&m, (aodvv2_msg_t *)msg.content.ptr, sizeof(m));
                    free(msg.content.ptr);

                    _send_rreq(&m.pkt, &m.next_hop);
                }
                break;

            case GNRC_NETAPI_MSG_TYPE_RCV:
                DEBUG("GNRC_NETAPI_MSG_TYPE_RCV\n");
                _receive((gnrc_pktsnip_t *)msg.content.ptr);
                break;
        }
    }

    /* Never reached */
    return NULL;
}
```
El código anterior representa el thread que escucha los mensajes y en el cual actualmente solo se pueden apreciar 2 caso posibles, por motivos de simplicidad se han removido los casos restantes para hacer énfasis en lo que realmente se quiere explicar.

- EL primer caso de la sentencia ```switch`` se ha explicado en el capitulo 12 de este documento
- El segundo caso es el objeto de estudio en este capitulo y trataremos de ver en detalle cada una de las tareas que se ejecutan para poder procesar de manera correcta el paquete entrante.

El porque es posible escuchar mensajes entrantes y ejecutar la funcion ```_receive```, se debe al registro a la red y al IPC que se hizo en el inicio del protocolo ```AODV``` visto en el capitulo 11.10

Teniendo claro que este es el caso que se dispara cuando tenemos mensajes entrantes a traves del protocolo UDP en el puerto especificado, pasemos a ver el contenido de la funcion que recibe el mensaje.

## 13.1 _receive
Como se dijo antes esta funcion se encarga de procesar el mensaje entrante a traves del protocolo UDP para el que nos hemos registrado.


```cpp
static void _receive(gnrc_pktsnip_t *pkt)
{
    assert(pkt != NULL && pkt->data != NULL && pkt->size > 0);
    /* Find sender address on IPv6 header */
    ipv6_addr_t sender;
    ipv6_hdr_t *ipv6_hdr = gnrc_ipv6_get_header(pkt);
    assert(ipv6_hdr != NULL);
    memcpy(&sender, &ipv6_hdr->src, sizeof(ipv6_addr_t));

    mutex_lock(&_reader_lock);
    aodvv2_rfc5444_handle_packet_prepare(&sender);
    if (rfc5444_reader_handle_packet(&_reader, pkt->data, pkt->size) != RFC5444_OKAY) {
        DEBUG("aodvv2: couldn't handle packet!\n");
    }
    mutex_unlock(&_reader_lock);

    gnrc_pktbuf_release(pkt);
}
```

El primer proposito de la funcion es tomar el paquete que recibe como argumento y extraer el header ```IPV6``` para asi obtener la dirección IP del originador del mensaje y eso se consigue con las siguientes lineas de código:
- Obtenemos el header IPV6.
- Asignamos a la variable IPV6 recién creada la información del originador del mensaje

```cpp
    ipv6_addr_t sender;
    ipv6_hdr_t *ipv6_hdr = gnrc_ipv6_get_header(pkt);
    assert(ipv6_hdr != NULL);
    memcpy(&sender, &ipv6_hdr->src, sizeof(ipv6_addr_t));
```

Lo siguiente es asignar la IP del originador del mensaje al paquete ```AODV``` que deseamos crear. mas adelante hacemos enfasis en esta funcion explicando como se hace la asignación, por el momento no es importante.
```cpp
 aodvv2_rfc5444_handle_packet_prepare(&sender);
 ```

Para deserializar el paquete que se ha recibido por el protocolo UDP, hacemos uso de la api que expone ```oonf_api```, y la cual entrega el control a las callbacks antes registradas para recibir la información deserializada.

Si tiene alguna duda hasta este punto, es buen momento de volver atrás y revisar la seccion 11.11.3 de este documento, la cual cubre el tema cerca del registro de los consmidores de mensajes y de direcciones.

Ya hemos dicho antes que vamos a procesar mensajes de requerimiento de ruta RREQ, asi que solo haremos énfasis en las callbacks relacionadas a este tipo de mensajes.

Las callback creadas para tal proceso están dentro del archivo ```rfc5444_reader.c``` y las cuales vamos a ver a continuacion.

En la seccion 11.11.3.2 se presentaron las siguientes estructuras para el registro de las callback.

```cpp
/*
 * Message consumer, will be called once for every message of
 * type RFC5444_MSGTYPE_RREQ that contains all the mandatory message TLVs
 */
static struct rfc5444_reader_tlvblock_consumer _rreq_consumer =
{
    .msg_id = RFC5444_MSGTYPE_RREQ,
    .block_callback = _cb_rreq_blocktlv_messagetlvs_okay,
    .end_callback = _cb_rreq_end_callback,
};

/*
 * Address consumer. Will be called once for every address in a message of
 * type RFC5444_MSGTYPE_RREQ.
 */
static struct rfc5444_reader_tlvblock_consumer _rreq_address_consumer =
{
    .msg_id = RFC5444_MSGTYPE_RREQ,
    .addrblock_consumer = true,
    .block_callback = _cb_rreq_blocktlv_addresstlvs_okay,
};
```

La primera callback en ejecutarse es la que corresponde al consumidor de bloque de direcciones, veamos su código y tratemos de entender cual es el proposito de leer el bloque de direcciones.

## 13.2  _cb_rreq_blocktlv_addresstlvs_okay
Esta callback permite deserializar unicamente el bloque de direcciones para asi evitar deserializar el mensaje completo, ahorrando costo computacional, lo que equivale a menos latencia entre mensajes.


```cpp
static enum rfc5444_result _cb_rreq_blocktlv_addresstlvs_okay(
        struct rfc5444_reader_tlvblock_context *cont)
{
    struct rfc5444_reader_tlvblock_entry *tlv;
    bool is_orig_node_addr = false;
    bool is_targ_node = false;

    DEBUG("rfc5444_reader: %s\n", netaddr_to_string(&nbuf, &cont->addr));

    /* handle OrigNode SeqNum TLV */
    tlv = _address_consumer_entries[RFC5444_MSGTLV_ORIGSEQNUM].tlv;
    if (tlv) {
        DEBUG("rfc5444_reader: RFC5444_MSGTLV_ORIGSEQNUM: %d\n",
              *tlv->single_value);

        is_orig_node_addr = true;
        packet_data.orig_node.addr = cont->addr;
        packet_data.orig_node.seqnum = *tlv->single_value;
    }

    /* handle TargNode SeqNum TLV */
    tlv = _address_consumer_entries[RFC5444_MSGTLV_TARGSEQNUM].tlv;
    if (tlv) {
        DEBUG("rfc5444_reader: RFC5444_MSGTLV_TARGSEQNUM: %d\n",
              *tlv->single_value);

        is_targ_node = true;
        packet_data.targ_node.addr = cont->addr;
        packet_data.targ_node.seqnum = *tlv->single_value;
    }

    if (!tlv && !is_orig_node_addr) {
        /* assume that tlv missing => targ_node Address */
        is_targ_node = true;
        packet_data.targ_node.addr = cont->addr;
    }

    if (!is_orig_node_addr && !is_targ_node) {
        DEBUG("rfc5444_reader: mandatory RFC5444_MSGTLV_ORIGSEQNUM TLV missing!\n");
        return RFC5444_DROP_PACKET;
    }

    /* handle Metric TLV */
    /* cppcheck: suppress false positive on non-trivially initialized arrays.
     *           this is a known bug: http://trac.cppcheck.net/ticket/5497 */
    /* cppcheck-suppress arrayIndexOutOfBounds */
    tlv = _address_consumer_entries[RFC5444_MSGTLV_METRIC].tlv;
    if (!tlv && is_orig_node_addr) {
        DEBUG("rfc5444_reader: Missing or unknown metric TLV.\n");
        return RFC5444_DROP_PACKET;
    }

    if (tlv) {
        if (!is_orig_node_addr) {
            DEBUG("rfc5444_reader: Metric TLV belongs to wrong address.\n");
            return RFC5444_DROP_PACKET;
        }
        DEBUG("rfc5444_reader: RFC5444_MSGTLV_METRIC val: %d, exttype: %d\n",
               *tlv->single_value, tlv->type_ext);
        packet_data.metric_type = tlv->type_ext;
        packet_data.orig_node.metric = *tlv->single_value;
    }
    return RFC5444_OKAY;
}
```
Ahora que tenemos la funcion de callback que se ejecuta cuando llega un paquete de requerimiento de ruta y lo que hace es leer el bloque de direcciones, veamos cual es el proceso para leer cada una de las direcciones disponibles.

Cabe destacar que la funcion de callback nos entrega le contexto de donde tenemos que leer el bloque de direcciones deserializado.








