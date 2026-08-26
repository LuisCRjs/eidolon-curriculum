# CS-M9 — Networking básico: DNS, TCP, TLS y HTTP

**Track:** Computer Science Foundations  
**Competencias:** D3.3; soporte D3.1, D3.2  
**Fase:** P0  
**Nivel objetivo:** Aplicado-profesional  
**Prerequisites:** PF-M1–PF-M9, CS-M1–CS-M8  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M9](../../02_curriculum/02_computer_science_foundations.md#cs-m9--networking-básico-dns-tcp-tls-y-http)  
**Status:** approved

El proceso A quiere enviar `{"event_id": "evt-001"}` al proceso B. Antes de hablar de frameworks, debe responder: ¿cómo encuentra A a B?, ¿qué bytes salen?, ¿dónde termina un mensaje?, ¿qué ocurre si el peer tarda o desaparece?, ¿y cómo sabe A si B aplicó el efecto?

> ¿Qué garantías ofrece cada capa y qué puede fallar entre un proceso que envía y otro que recibe?

```text
process A
↓ application data → serialization → bytes
↓ transport
↓ network / link substrate
↓ transport
process B
```

Después:

```text
application intent
↓ HTTP request
↓ protected/plain transport según scheme
↓ HTTP response o fallo
↓ application decision
```

CS-M9 no construye backend ni administra redes. Enseña a localizar una garantía o un fallo en la capa correcta.

## Resultados de aprendizaje

Al terminar podrás:

- distinguir host, process, client/server role y endpoint;
- explicar IPv4, IPv6, loopback, localhost, bind address y port a nivel práctico;
- describir un socket como resource del OS y gestionar su lifecycle;
- convertir application data en bytes y definir framing explícito;
- explicar por qué TCP stream, `send`/`recv`, packet y message no son equivalentes;
- implementar un protocolo length-prefixed local con reads exactos y límites;
- diferenciar latency, bandwidth, throughput y RTT;
- explicar DNS resolution/cache y separar DNS de connect/HTTP failures;
- delimitar garantías de TCP, UDP, flow control y application backpressure;
- diseñar timeout budgets y clasificar ambiguous outcomes;
- decidir si un retry es seguro desde method/effect/idempotency key;
- interpretar request, response, URL, method, status, headers y body;
- separar transport success, HTTP status, JSON validity y domain validity;
- explicar TLS sin atribuirle validación del contenido de dominio;
- ejecutar TCP/UDP/HTTP experiments solo en loopback y cerrarlos limpiamente;
- diagnosticar por capas con evidencia y sin registrar payload sensible;
- preservar source data y tratar respuestas remotas como derived evidence.

## Cómo estudiar este módulo

1. Dibuja las capas antes de culpar a “la red”.
2. Escribe los bytes y el framing antes de llamar `recv`.
3. Declara connect/read/overall timeout por separado.
4. Pregunta qué sabe el client después de cada fallo.
5. Prueba siempre en loopback y port efímero.
6. Cierra server, client, threads y sockets incluso ante excepción.
7. Usa payload sintético; nunca captures tokens o conversaciones reales.

### Convenciones

- **Ejemplo ejecutable:** autónomo, loopback-only, acotado y con cleanup.
- **Failure case controlado:** reproduce una propiedad sin depender de Internet.
- **Fragmento:** muestra wire shape o idea; requiere contexto.
- **Comando Linux/Unix-like:** se etiqueta; no define semántica portable.
- **Modelo educativo:** simplifica protocolos, no reemplaza estándares ni production hardening.

Baseline: Python 3.14 y standard library. Ports, timings, address order y mensajes exactos de error dependen del entorno; se comprueban propiedades estables.

---

## 1. Capas, hosts y roles

Un modelo útil, no dogmático:

```text
Application   HTTP / protocolo propio pequeño
Transport     TCP / UDP
Network       IP + routing
Link          frames sobre el medio local
```

OSI y TCP/IP son modelos de referencia; implementaciones reales no encajan siempre en una tabla escolar perfecta.

Un **host** es un sistema participante en una red. Un **process** es una ejecución en ese host. Un host puede ejecutar muchos processes y tener varias interfaces/IP addresses.

**Client** y **server** son roles: client inicia una interacción; server escucha/acepta/responde. El mismo programa puede ser client en una frontera y server en otra. **Peer** enfatiza dos participantes sin fijar permanentemente esos roles.

### Capa

Clasifica: resolver un nombre, aceptar conexión, parsear JSON y decidir si un Event es válido. ¿Qué capa conoce cada decisión?

## 2. IP, loopback, localhost y ports

Una IP address identifica una interfaz/localización de routing dentro del modelo de network layer.

- IPv4 usa 32 bits; `192.0.2.1` pertenece a un rango de documentación.
- IPv6 amplía enormemente el espacio; `2001:db8::1` es ejemplo documental.
- `127.0.0.1` e `::1` son loopback: tráfico hacia el propio host.
- `localhost` es un hostname normalmente resuelto a loopback; puede producir IPv4, IPv6 o ambas según configuración.

Una address private se reserva para contextos de routing privado; una public puede ser globalmente routable según asignación/rutas. Ninguna etiqueta garantiza reachability o seguridad: NAT, firewall y policy influyen.

Un **port** es un número de transporte de 16 bits (`0`–`65535`). Algunos rangos/números tienen convenciones well-known, pero no memorizamos un catálogo. En la práctica, TCP y UDP mantienen namespaces de ports separados. El modelo de endpoint debe incluir al menos protocolo + address + port:

```text
TCP 127.0.0.1:8000
```

El port 0 se usa al hacer `bind` para pedir al OS un port efímero; no es el port final al que conecta el client.

**Ejemplo ejecutable:**

```python
import socket

addresses = socket.getaddrinfo(
    "localhost",
    80,
    type=socket.SOCK_STREAM,
)
assert addresses
assert all(len(item) == 5 for item in addresses)
print("localhost resolved to at least one address")
```

No fijes address/order: dependen del host.

### Endpoint

¿Por qué `localhost:8000` es insuficiente si no sabes transport protocol y address family?

## 3. Bind address y exposición

Un IPv4 server ligado a `127.0.0.1` escucha solo por loopback. Ligarlo a `0.0.0.0` solicita escuchar en todas las interfaces IPv4 locales aplicables; `0.0.0.0` no es normalmente el destino que un remote client usa para conectar.

```text
127.0.0.1   loopback-only
0.0.0.0     all local IPv4 interfaces aplicables
```

Para IPv6, `::` y el comportamiento dual-stack dependen de plataforma/configuración. Escuchar en todas las interfaces aumenta exposición; no lo uses por comodidad sin threat boundary y firewall policy. Localhost-only reduce exposición de red, pero no elimina procesos locales maliciosos, browser risks ni egress futuro.

### Endpoint

Un server imprime `0.0.0.0:8000`. ¿Qué significa para bind y qué address usarías desde el mismo host para probarlo?

## 4. Socket y lifecycle

Un socket no es “IP + port”. Es una abstracción/resource del OS para comunicación:

```text
Python socket object
↓
OS socket / descriptor-like resource
↓
network stack
```

Server TCP:

```text
socket → bind → listen → accept → recv/send → close
```

Client TCP:

```text
socket → connect → send/recv → close
```

`accept()` produce otro connected socket; el listening socket sigue representando el listener. Ambos tienen owners/lifecycles. Usa context managers, timeout y shutdown coordinado como en CS-M7/CS-M8.

### Ownership

¿Quién cierra listening socket, accepted socket, client socket y server thread en un test local?

## 5. Bytes, Unicode y serialization

La red transporta bytes, no dicts ni dataclasses:

```text
application object
↓ JSON serialization
str
↓ UTF-8 encode
bytes
↓ network
bytes
↓ UTF-8 decode + JSON parse + schema/domain validation
application value
```

**Ejemplo ejecutable:**

```python
import json

record = {"event_id": "evt-001", "text": "Llegué"}
text = json.dumps(record, ensure_ascii=False, separators=(",", ":"))
payload = text.encode("utf-8")
decoded = json.loads(payload.decode("utf-8"))

assert isinstance(payload, bytes)
assert decoded == record
print("serialization boundary: PASS")
```

Transportar bytes correctamente no valida schema, provenance ni domain truth.

### Bytes

¿En qué paso puede fallar encoding, JSON parsing y domain validation? No los colapses en “network error”.

## 6. TCP es stream; framing crea mensajes

TCP entrega un ordered byte stream dentro de una conexión. No conserva llamadas de aplicación:

```text
send(A) + send(B)
≠
recv() devuelve exactamente A y luego exactamente B
```

Un `recv(4096)` puede devolver menos, exactamente esa cantidad, combinar bytes disponibles o indicar EOF con `b""`. Un `send()` low-level puede aceptar solo parte del buffer y retorna el count. `sendall()` intenta enviar todo el buffer o lanza; no informa que el peer procesó el mensaje.

**Framing** define boundaries. Opciones: delimiter, fixed length, length prefix o un protocolo superior como HTTP. Newline framing sirve si encoding/escaping y tamaño están definidos. Length prefix permite bytes arbitrarios, pero requiere límite.

Packet tampoco es application message: un message puede abarcar múltiples packets/segments y sus bytes pueden agruparse de otra forma entre capas. No construyas framing con packet boundaries.

### Framing

El payload contiene un newline válido dentro de un string. ¿Qué contrato adicional necesita newline framing? ¿Cuándo length prefix es más claro?

## 7. Length-prefix protocol educativo

Usaremos 4 bytes unsigned en network byte order (`!I`) seguidos del payload. `recv_exact` acumula hasta completar o reporta EOF. El límite impide que un prefix corrupto reserve memoria sin control.

**Ejemplo ejecutable:**

```python
import socket
import struct
from threading import Thread

MAX_FRAME = 65_536


def recv_exact(sock: socket.socket, size: int) -> bytes:
    chunks: list[bytes] = []
    remaining = size
    while remaining:
        chunk = sock.recv(remaining)
        if not chunk:
            raise EOFError("connection closed mid-frame")
        chunks.append(chunk)
        remaining -= len(chunk)
    return b"".join(chunks)


def send_frame(sock: socket.socket, payload: bytes) -> None:
    if len(payload) > MAX_FRAME:
        raise ValueError("frame too large")
    sock.sendall(struct.pack("!I", len(payload)) + payload)


def recv_frame(sock: socket.socket) -> bytes:
    (size,) = struct.unpack("!I", recv_exact(sock, 4))
    if size > MAX_FRAME:
        raise ValueError("frame too large")
    return recv_exact(sock, size)


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as listener:
    listener.settimeout(2)
    listener.bind(("127.0.0.1", 0))
    listener.listen(1)
    host, port = listener.getsockname()

    def echo_two() -> None:
        connection, _ = listener.accept()
        with connection:
            connection.settimeout(2)
            for _ in range(2):
                send_frame(connection, recv_frame(connection))

    server = Thread(target=echo_two, name="framed-echo")
    server.start()
    with socket.create_connection((host, port), timeout=2) as client:
        client.settimeout(2)
        send_frame(client, "uno".encode("utf-8"))
        send_frame(client, "dos".encode("utf-8"))
        replies = [recv_frame(client).decode("utf-8") for _ in range(2)]

    server.join(timeout=2)
    assert not server.is_alive()
    assert replies == ["uno", "dos"]

print("framed TCP echo: PASS")
```

El test usa loopback/IPv4/port efímero. Demuestra framing y lifecycle, no Internet ni durability/application acknowledgement.

### Modifica

Envía un frame vacío y uno que contenga Unicode/newline. ¿Qué properties deben conservarse?

## 8. Truncamiento, close y estados de conexión

Si EOF llega antes de completar header/payload, el frame está truncado. No inventes bytes. El client debe decidir si reporta, descarta derived partial state o reintenta según effect semantics.

Modelo pedagógico de lifecycle:

```text
CLOSED → CONNECTING → ESTABLISHED → CLOSING → CLOSED
```

No replica el state machine TCP completo. Un cierre orderly y un reset/broken pipe son outcomes distintos. Half-close permite cerrar una direction de comunicación, pero queda como extensión.

`close()` local libera el recurso y participa en el protocolo de cierre; no prueba que el peer aplicó un effect.

### Failure

`sendall` retornó y luego la connection se reseteó. ¿Qué sabe el sender sobre bytes aceptados localmente y application effect remoto?

## 9. Latency, bandwidth, throughput, RTT y MTU

- **latency:** tiempo desde un inicio hasta un resultado definido;
- **bandwidth:** capacidad nominal/efectiva de transferencia por tiempo;
- **throughput:** trabajo/data completado por tiempo;
- **RTT:** ida y vuelta medida en una frontera, no toda request latency.

```text
request latency
≈ DNS + connect + TLS + server work + transfer + queuing
```

Los componentes pueden solaparse/reutilizarse; es un budget, no identidad física universal. Alta bandwidth no implica baja latency. Throughput puede quedar limitado por server, contention o application queue.

MTU limita tamaño de unidades en un link/path a nivel introductorio. Datos mayores pueden dividirse en capas inferiores; no uses MTU como message boundary. Routing puede atravesar routers/hops; no estudiamos BGP/OSPF.

### Workload

Compara: enlace satelital de alta bandwidth/alto RTT y enlace local de menor bandwidth/bajo RTT. ¿Cuál responde antes a un request pequeño?

## 10. DNS y resolución

DNS relaciona names con información de naming, incluida network addresses. Una resolución conceptual puede involucrar resolver local/cache, recursive resolver y authoritative data. No es una tabla instantánea global.

```text
hostname
↓ resolver + caches
una o varias addresses
```

Records/caches tienen lifetimes; un cambio no aparece simultáneamente en todos lados. Un name puede resolver a múltiples addresses y cambiar con tiempo.

Failure modes:

- `socket.gaierror`: name/address resolution;
- timeout del resolver;
- stale/wrong address;
- resolución correcta pero connect falla después.

DNS failure ≠ connection refused ≠ TLS failure ≠ HTTP 500.

### Capa

`getaddrinfo` retorna addresses, pero ninguna acepta conexión. ¿Qué capa funcionó y cuál falla después?

## 11. TCP: garantías y límites

TCP ofrece un connection-oriented, reliable, ordered byte stream dentro de su modelo:

- sequence numbers y acknowledgements permiten ordenar/detectar transporte faltante;
- retransmission intenta recuperar pérdida;
- retransmitted transport data no debe convertirse por eso en bytes duplicados del stream;
- flow control adapta al receiver;
- congestion control adapta envío a network conditions.

No garantiza:

- que la conexión nunca falle;
- que exista un message boundary;
- que el peer haya parseado/validado/committed;
- que un retried application request no duplique effects;
- application-level backpressure.

El three-way handshake (`SYN`, `SYN-ACK`, `ACK`) explica que establecer conexión tiene RTT/estado. No memorizamos flags. Retransmission puede aumentar latency aunque ordering se conserve.

### Semantics

El server TCP recibió todos los bytes y respondió ACKs de transporte. ¿Qué falta para afirmar que un Event fue committed?

## 12. Flow control no sustituye backpressure

TCP puede evitar desbordar buffers de transporte del receiver. La aplicación aún puede aceptar bytes/requests más rápido de lo que procesa jobs:

```text
socket/read
↓ bounded application queue
workers
↓ result
```

CS-M8 ya enseñó bounded queue. Network flow control y application admission/capacity protegen invariantes distintos.

Persistent connections reducen setup repetido, pero agregan lifecycle/pooling/idle timeout. No asumas “una TCP connection por HTTP request” como regla universal.

### Backpressure

El socket sigue readable, pero la Queue está llena. ¿Debe el reader seguir materializando requests ilimitados?

## 13. UDP: datagrams con otro contrato

UDP conserva datagram boundaries y no establece un reliable ordered stream. Datagramas pueden perderse, duplicarse o reordenarse. Esto no lo convierte en “junk” ni “TCP rápido”: puede servir cuando application semantics toleran pérdida o implementan su propia policy.

**Ejemplo ejecutable local:**

```python
import socket
from threading import Thread

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as server_socket:
    server_socket.bind(("127.0.0.1", 0))
    server_socket.settimeout(2)
    endpoint = server_socket.getsockname()

    def echo_datagram() -> None:
        payload, sender = server_socket.recvfrom(1024)
        server_socket.sendto(payload, sender)

    server = Thread(target=echo_datagram, name="udp-echo")
    server.start()
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as client:
        client.settimeout(2)
        client.sendto(b"synthetic", endpoint)
        reply, _ = client.recvfrom(1024)

    server.join(timeout=2)
    assert not server.is_alive()
    assert reply == b"synthetic"

print("local UDP datagram: PASS")
```

El éxito local no demuestra delivery guarantee general.

### Workload

¿Qué requirements justificarían UDP? Responde con tolerance a loss/order y recovery, no solo “velocidad”.

## 14. Timeouts, deadlines y failure classification

Timeout es parte del contrato. Distingue:

- resolution timeout;
- connect timeout;
- read timeout;
- overall operation deadline.

Un overall budget limita cuánto puede gastar la operación completa. Dar el mismo timeout completo a cada retry puede excederlo.

```text
symptom: no response
↓
resolution? connect? TLS? write? read? HTTP? application?
```

Connection refused suele indicar que el target respondió que no había listener compatible en ese endpoint. Timeout no prueba lo mismo: firewall, packet loss, server lento o read sin respuesta son hipótesis distintas.

### Diagnose

¿Qué evidencia distingue `gaierror`, `ConnectionRefusedError`, `TimeoutError` y HTTP 503?

## 15. Retry, ambiguous outcome e idempotency

Timeline central:

```text
client sends create(operation_id=op-001)
server commits effect
response is lost/delayed
client times out
```

El outcome es **ambiguo**: el client no sabe si request no llegó, falló antes de commit, committed o solo perdió response.

Retry repite una operación. Debe responder:

- ¿el fallo es transient, permanent o contractual?
- ¿queda budget?
- ¿el effect es idempotent?
- ¿existe stable operation/idempotency key?

Una idempotency key es un patrón de aplicación: el server recuerda el logical operation y evita duplicate effect bajo su contrato. No es magia estándar universal ni exactly-once distribuido. Backoff creciente y jitter pueden reducir retry storms; aquí solo introducimos la intuición.

### Retry

¿Por qué `POST create` sin operation ID no debe reintentarse ciegamente después de read timeout?

## 16. HTTP: request/response sobre una frontera

HTTP aporta semántica application-level:

```text
request:  method target version + headers + optional body
response: status version + headers + optional body
```

Un HTTP 500 puede viajar correctamente sobre TCP/TLS: transport success y application status son capas distintas.

URL práctica:

```text
scheme://host:port/path?query#fragment
```

El fragment normalmente lo interpreta el client y no se envía como request target al server.

Lifecycle conceptual de un request:

```text
build request → resolve → connect → optional TLS
→ send → wait/read → parse HTTP → application decision
```

Cada flecha agrega un failure/timeout boundary. Una persistent connection puede omitir resolution/connect/TLS para requests posteriores, pero no elimina sus lifecycles.

**Ejemplo ejecutable:**

```python
from urllib.parse import urlsplit

parts = urlsplit("https://example.test:8443/events?tag=home#latest")
assert parts.scheme == "https"
assert parts.hostname == "example.test"
assert parts.port == 8443
assert parts.path == "/events"
assert parts.query == "tag=home"
assert parts.fragment == "latest"
print("URL components: PASS")
```

### Semantics

¿Qué componentes determinan connection target y cuál no cruza normalmente la frontera HTTP?

## 17. Methods, safety e idempotency semántica

- `GET`: retrieve; safe e idempotent por semántica;
- `HEAD`: como GET sin response body semántico; safe/idempotent;
- `POST`: procesa representación; no necesariamente idempotent;
- `PUT`: reemplaza/define estado del target; idempotent por intención;
- `PATCH`: modificación parcial; no necesariamente idempotent;
- `DELETE`: idempotent por efecto intencional, aunque responses repetidas puedan diferir.

Safe significa que el client no solicita cambio de estado; no es security guarantee. Una implementación puede violar semántica. No conviertas methods en CRUD rígido; Backend Track diseñará contracts.

### Retry

¿Por qué “DELETE es idempotent” no obliga a que la segunda response tenga el mismo status/body?

## 18. Status codes y diagnóstico

Categorías:

- 1xx informational;
- 2xx successful HTTP outcome (`200`, `201`, `204`);
- 3xx redirection;
- 4xx request/client-side contract issue (`400`, `404`, `409`);
- 5xx server-side failure (`500`, `503`).

Un 404 prueba que obtuviste una HTTP response; no significa DNS failure ni server unreachable. Un 500 prueba que la interacción alcanzó una capa capaz de emitir HTTP; no es connection error. Un 503 puede ser retryable según method, headers, budget y policy; no “retry todo 5xx”.

### Capa

Ordena desde capa inferior: DNS failure, refused connect, TLS hostname failure, HTTP 404, JSON schema invalid.

## 19. Headers, body, Content-Type y JSON

Headers transportan metadata. Ejemplos: `Content-Type`, `Content-Length`, `Accept`, `Retry-After`. `Authorization` existe, pero auth queda para Backend/Security.

```text
body bytes + Content-Type → interpretación declarada
```

`application/json` declara media type; no garantiza JSON válido ni domain-valid. `Content-Length` es una forma de delimitar body. HTTP también posee otros framing mechanisms; chunked transfer se menciona, no se implementa.

```text
transport bytes OK
↓ Content-Type compatible
↓ UTF-8/JSON parse
↓ schema validation
↓ domain validity
```

No registres body completo por default: puede contener tokens, cookies, conversaciones o datos personales.

### Privacy

Diseña un log de request con method, redacted path, status, elapsed y operation ID, sin body ni secrets.

## 20. HTTP versions y persistent connections

Panorama, no internals:

- HTTP/1.1 permite persistent connections y framing propio;
- HTTP/2 multiplexa streams sobre una connection y usa framing binario;
- HTTP/3 usa QUIC sobre UDP.

HTTP no equivale universalmente a una TCP connection nueva por request. Proxies y pools pueden reutilizar conexiones. CS-M9 razona sobre request semantics sin implementar esos protocolos.

## 21. TLS y HTTPS

TLS aporta, bajo configuración/validación correctas:

- confidentiality del tránsito;
- integrity del canal;
- peer authentication habitual mediante certificate, hostname validation y trust chain.

```text
HTTPS ≈ HTTP sobre una sesión/transporte protegido por TLS
```

TLS no decide que el JSON es correcto, que el Claim es verdadero ni que el remote service debe recibir ese dato. `http://` envía sin esa protección; HTTP local puede ser aceptable en un experimento loopback, no una decisión automática para remote production.

### Semantics

Un certificate/hostname válido protege el canal. ¿Qué validaciones de payload y domain siguen pendientes?

## 22. Proxy, NAT y firewall: contexto mínimo

```text
client → proxy → server
```

Un proxy agrega otra frontera observable. NAT ayuda a mapear addresses/connections y explica por qué private addresses no suelen ser directly reachable desde Internet. Un firewall puede filtrar traffic. Ninguno debe usarse como explicación automática.

Networked systems admiten **partial failure**: client puede estar sano mientras server, una route, DNS cache o una dependency no lo está. “Todo arriba”/“todo abajo” rara vez describe evidencia suficiente.

Ante “no conecta”, distingue:

- name no resuelve;
- route/address incorrecta;
- port sin listener;
- firewall/filter timeout;
- TLS negotiation/validation;
- HTTP status;
- application error.

No enseñamos port forwarding, load balancers ni cloud networking.

## 23. Debugging por capas y herramientas

```text
symptom
↓ identify layer
↓ collect evidence
↓ test smallest boundary
↓ move upward
```

| Herramienta | Pregunta | Límite |
|---|---|---|
| `ping` | ¿hay respuesta ICMP básica? | bloqueo ICMP; no prueba port/service |
| `curl -v URL` | ¿qué request/response HTTP observa el client? | no explica por sí solo DNS/TCP/TLS cause |
| `ss -ltn` | ¿qué TCP listeners muestra Linux? | Linux-specific |
| `lsof -i` | ¿qué sockets asocia Unix-like a processes? | opcional/permisos |
| `dig`/`nslookup` | ¿qué DNS answer observa la herramienta? | disponibilidad/cache/contexto |
| `traceroute`/`tracert` | ¿qué hops responden? | filtros/asymmetry; no causalidad completa |

Packet capture con `tcpdump`/Wireshark es preview. Puede exponer secrets; usa loopback y synthetic payload. En Windows usa herramientas equivalentes y no proyectes `ss`/`lsof`.

### Diagnose

`ping` falla, pero `curl` recibe 200. ¿Por qué no hay contradicción?

## 24. HTTP server local con standard library

**Ejemplo ejecutable:** loopback + port efímero + shutdown.

```python
import json
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from threading import Thread
from urllib.request import urlopen


class HealthHandler(BaseHTTPRequestHandler):
    def do_GET(self) -> None:
        if self.path != "/health":
            self.send_error(404)
            return
        body = json.dumps({"status": "ok"}).encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format: str, *args: object) -> None:
        pass


server = ThreadingHTTPServer(("127.0.0.1", 0), HealthHandler)
thread = Thread(target=server.serve_forever, name="local-http")
thread.start()
try:
    host, port = server.server_address
    with urlopen(f"http://{host}:{port}/health", timeout=2) as response:
        body = json.load(response)
        assert response.status == 200
        assert response.headers.get_content_type() == "application/json"
        assert body == {"status": "ok"}
finally:
    server.shutdown()
    server.server_close()
    thread.join(timeout=2)

assert not thread.is_alive()
print("local HTTP request: PASS")
```

`ThreadingHTTPServer` solo simplifica el experimento; no es production backend.

### Modifica

Solicita `/missing` y captura `urllib.error.HTTPError`. Comprueba `code == 404`; no lo clasifiques como network unreachable.

## 25. Latency y read timeout local

**Failure case ejecutable:** el server demora body más que el client timeout. La exception demuestra timeout de espera, no que el server no recibió el request.

```python
from http.client import HTTPConnection
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from threading import Thread
from time import sleep


class SlowHandler(BaseHTTPRequestHandler):
    def do_GET(self) -> None:
        sleep(0.08)
        body = b"late"
        self.send_response(200)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        try:
            self.wfile.write(body)
        except (BrokenPipeError, ConnectionResetError):
            pass

    def log_message(self, format: str, *args: object) -> None:
        pass


server = ThreadingHTTPServer(("127.0.0.1", 0), SlowHandler)
server.daemon_threads = False
thread = Thread(target=server.serve_forever)
thread.start()
connection = HTTPConnection(*server.server_address, timeout=0.02)
try:
    connection.request("GET", "/slow")
    try:
        connection.getresponse()
    except TimeoutError:
        pass
    else:
        raise AssertionError("read timeout was not observed")
finally:
    connection.close()
    server.shutdown()
    server.server_close()
    thread.join(timeout=2)

assert not thread.is_alive()
print("ambiguous read timeout observed: PASS")
```

Loopback delay is synthetic, not Internet latency.

### Failure

¿Qué evidence adicional necesitarías para saber si el handler ejecutó un effect antes del timeout?

## 26. Idempotency experiment y ambiguous response

El server registra `Idempotency-Key`. La primera request commits y demora response; el client agota timeout. El retry con la misma key obtiene el resultado sin duplicar effect.

**Modelo educativo ejecutable:**

```python
import json
from http.client import HTTPConnection
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from queue import Queue
from threading import Event, Lock, Thread


class OperationHandler(BaseHTTPRequestHandler):
    committed: set[str] = set()
    effect_count = 0
    lock = Lock()
    first_committed = Event()
    release_first_response = Event()

    def do_POST(self) -> None:
        if self.path != "/operations":
            self.send_error(404)
            return
        key = self.headers.get("Idempotency-Key")
        if not key:
            self.send_error(400)
            return
        length = int(self.headers.get("Content-Length", "0"))
        payload = json.loads(self.rfile.read(length).decode("utf-8"))
        if payload.get("operation_id") != key:
            self.send_error(409)
            return

        with type(self).lock:
            duplicate = key in type(self).committed
            if not duplicate:
                type(self).committed.add(key)
                type(self).effect_count += 1
                type(self).first_committed.set()

        if not duplicate:
            type(self).release_first_response.wait(timeout=2)
        body = json.dumps({"operation_id": key, "duplicate": duplicate}).encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        try:
            self.wfile.write(body)
        except (BrokenPipeError, ConnectionResetError):
            pass

    def log_message(self, format: str, *args: object) -> None:
        pass


def post(host: str, port: int, timeout: float) -> tuple[int, dict[str, object]]:
    body = json.dumps({"operation_id": "op-001"}).encode("utf-8")
    connection = HTTPConnection(host, port, timeout=timeout)
    try:
        connection.request(
            "POST",
            "/operations",
            body=body,
            headers={
                "Content-Type": "application/json",
                "Content-Length": str(len(body)),
                "Idempotency-Key": "op-001",
            },
        )
        response = connection.getresponse()
        return response.status, json.loads(response.read().decode("utf-8"))
    finally:
        connection.close()


OperationHandler.committed = set()
OperationHandler.effect_count = 0
OperationHandler.first_committed = Event()
OperationHandler.release_first_response = Event()
server = ThreadingHTTPServer(("127.0.0.1", 0), OperationHandler)
server.daemon_threads = False
thread = Thread(target=server.serve_forever)
thread.start()
host, port = server.server_address
first_outcome: Queue[Exception] = Queue()


def first_attempt() -> None:
    try:
        post(host, port, timeout=0.05)
    except Exception as error:
        first_outcome.put(error)


try:
    client = Thread(target=first_attempt, name="ambiguous-client")
    client.start()
    assert OperationHandler.first_committed.wait(timeout=1)
    client.join(timeout=1)
    assert not client.is_alive()
    assert isinstance(first_outcome.get(timeout=1), TimeoutError)

    status, response = post(host, port, timeout=1)
    assert status == 200
    assert response == {"operation_id": "op-001", "duplicate": True}
    assert OperationHandler.effect_count == 1
finally:
    OperationHandler.release_first_response.set()
    server.shutdown()
    server.server_close()
    thread.join(timeout=2)

assert not thread.is_alive()
print("idempotent retry: PASS")
```

Esto no implementa exactly-once distribuido: state vive en memoria de un process y desaparece al reiniciar. Enseña la relation entre ambiguous outcome y stable key.

### Retry

¿Qué falla si el server pierde su set de keys antes del retry? ¿Qué track futuro deberá diseñar persistence/transactions?

## 27. EIDOLON: local-first y data boundary

Incluso un futuro frontend local hablando con un backend local cruza:

```text
process → serialization → loopback socket → HTTP → process
```

Loopback no es Internet, pero sí una process/protocol boundary con ports, timeouts y validation.

Una frontera remota futura añade routing, TLS, provider availability y privacy. Antes de enviar:

```text
¿qué bytes salen del host?
¿son necesarios?
¿incluyen source o conversaciones sensibles?
¿qué retention/policy tiene el peer?
```

Disciplina EIDOLON:

```text
source evidence
→ request derived from source
→ remote response
→ derived result + provenance
```

La response no sobrescribe source ni se convierte en truth por viajar por HTTPS.

### Privacy

Diseña un payload sintético mínimo y enumera qué campos no enviarías a un remote provider sin policy explícita.

## 28. Catálogo de failure cases

| Creencia/fallo | Qué rompe | Corrección |
|---|---|---|
| `recv()` = un message | stream puede fragmentar/combinar | framing + accumulation |
| `send()` = todos los bytes | retorno parcial posible | loop o `sendall`; aún no es app ACK |
| packet = message | capas/boundaries distintas | protocol framing |
| TCP = peer processed | solo transporte | application response/commit protocol |
| timeout = request no ocurrió | outcome puede ser ambiguo | operation ID + status/recovery policy |
| retry POST ciego | duplicate effect | semantics/idempotency key |
| UDP = unreliable junk | ignora workloads adecuados | elegir por requirements |
| 404 = unreachable | server sí respondió HTTP | clasificar HTTP/application |
| 500 = connect error | response HTTP recibida | separar transport/status |
| bandwidth alta = latency baja | métricas distintas | medir workload/frontera |
| localhost = `0.0.0.0` | resolución vs bind wildcard | loopback-only vs all interfaces |
| DNS name = una IP eterna | caches/multiple addresses/change | resolve/observe por operación/policy |
| retry todas las exceptions | permanent/contract errors empeoran | clasificación + budget/backoff |
| bind all interfaces sin awareness | exposición accidental | loopback default/threat model |
| loggear body completo | fuga de data/secrets | metadata/IDs/redaction |
| TCP flow control = app capacity | backlog lógico ilimitado | bounded queue/admission |

## 29. Ejercicios guiados

### Guiado 1 — Host, process y endpoint

Dibuja un host con dos processes: server en TCP loopback port efímero y client. La solución distingue system, PID/role y endpoint protocol+address+port.

### Guiado 2 — Loopback vs remote

Predice reachability para bind `127.0.0.1` y `0.0.0.0`. Explica por qué el segundo aumenta exposición y no es destination address.

### Guiado 3 — Construye listener

Usa `socket(AF_INET, SOCK_STREAM)`, bind loopback/0, timeout y `listen`. Obtén `getsockname`; el criterio es port final > 0 y cleanup con `with`.

### Guiado 4 — Construye client

Usa `create_connection` con timeout hacia el listener del guiado 3. Owner espera server thread y cierra ambos sockets.

### Guiado 5 — Bytes y UTF-8

Ejecuta sección 5 con texto Unicode. Verifica round-trip y distingue bytes de object.

### Guiado 6 — Boundary problem

Divide artificialmente `b"abcdef"` en `b"ab"`, `b"cde"`, `b"f"`. Reconstruye sin asumir calls 1:1.

### Guiado 7 — Framing

Ejecuta sección 7 con dos frames. Trunca el segundo y exige `EOFError`; nunca retorna partial message como válido.

### Guiado 8 — Timeout

Ejecuta sección 25. Identifica read timeout y explica por qué outcome no es “definitely failed”.

### Guiado 9 — Refused vs DNS

Clasifica `socket.gaierror` de resolution y `ConnectionRefusedError` de connect. No dependas del texto exacto ni de Internet para el test final.

### Guiado 10 — Request/response

Escribe wire shape mínimo de GET y response 200. Señala method, target, status, headers y body.

### Guiado 11 — HTTP server

Ejecuta sección 24. Verifica loopback, ephemeral port, JSON Content-Type y shutdown.

### Guiado 12 — `curl`

Mientras un server local controlado está activo, ejecuta `curl -v http://127.0.0.1:<PORT>/health`. Separa request lines, response status y connection details; redáctalos antes de compartir.

### Guiado 13 — Status codes

Solicita `/missing`, captura `HTTPError` y verifica 404. Explica por qué DNS/TCP funcionaron.

### Guiado 14 — JSON over HTTP

Envía synthetic JSON con `Content-Type`, lee según `Content-Length`, parsea y valida `operation_id` por separado.

### Guiado 15 — Retry controlado

Define máximo attempts, overall deadline y error retryable. La solución no reintenta 400/schema invalid.

### Guiado 16 — Ambiguous outcome

Traza commit antes de response timeout en sección 26. Enumera cuatro outcomes posibles desde el client.

### Guiado 17 — Idempotency key

Ejecuta sección 26. El segundo request usa la misma key; `effect_count` sigue en 1.

### Guiado 18 — Mide latency

Mide con `perf_counter` request local normal y delayed. Reporta distribución, no llama Internet latency al loopback.

### Guiado 19 — Privacy boundary

Redacta un request log sintético sin body, token/cookie ni conversation. Incluye operation ID, method, status, elapsed y layer.

### Guiado 20 — Debugging por capas

Dado “no conecta”, entrega árbol: resolve → endpoint/listener → connect → TLS → HTTP → JSON/application. Cada paso tiene una observación mínima y no cambia código antes de evidence.

## 30. Ejercicios independientes

1. Dibuja application/transport/network/link para un request.
2. Separa host, process, role y endpoint.
3. Compara IPv4/IPv6 y loopback sin asumir address order.
4. Explica private/public/NAT sin llamar private “seguro”.
5. Abre listener loopback en port efímero y ciérralo.
6. Explica por qué accepted socket y listener tienen lifecycles distintos.
7. Serializa synthetic Event a UTF-8 bytes y valida al volver.
8. Implementa `recv_exact` con EOF controlado.
9. Implementa delimiter framing con límite de tamaño.
10. Compara delimiter y length prefix para JSON.
11. Simula partial send con una función fake que acepta chunks.
12. Explica packet/frame/segment/message sin igualarlos.
13. Calcula throughput y diferencia de latency/RTT.
14. Diseña timeout budget DNS/connect/read/overall.
15. Clasifica cinco failures por capa.
16. Resuelve localhost con `getaddrinfo` sin fijar una IP.
17. Explica DNS cache/stale answer y siguiente failure posible.
18. Dibuja handshake/cierre TCP a nivel conceptual.
19. Explica retransmission/order sin app acknowledgement.
20. Diseña application backpressure después de socket read.
21. Ejecuta datagram echo y documenta límites.
22. Elige TCP/UDP para tres requirements.
23. Parse URL y separa fragment.
24. Diseña method semantics para una operación sintética.
25. Distingue safety de idempotency.
26. Clasifica 201/204/400/404/409/500/503.
27. Valida Content-Type antes de JSON parse.
28. Distingue JSON valid de domain valid.
29. Explica persistent connection/HTTP versions panorámicamente.
30. Explica qué protege TLS y qué no.
31. Compara bind loopback/all interfaces.
32. Usa `curl -v` solo contra server local.
33. Observa listener con herramienta de plataforma.
34. Diseña retry con stable operation ID.
35. Simula lost response y duplicate-safe retry.
36. Explica retry storm/backoff sin implementar librería.
37. Diseña log redacted de network operation.
38. Reconstruye una failure timeline con IDs.
39. Preserva source frente a remote response.
40. Escribe data-boundary checklist antes de egress.

## 31. Preguntas conceptuales

1. ¿Qué diferencia existe entre host, process, role y endpoint?
2. ¿Qué problema resuelve un port?
3. ¿Por qué endpoint práctico incluye transport protocol?
4. ¿Qué es realmente un socket?
5. ¿Qué diferencia listener de connected socket?
6. ¿Por qué localhost no implica una única IP?
7. ¿Qué diferencia loopback de `0.0.0.0`?
8. ¿Qué bytes produce JSON+UTF-8?
9. ¿Por qué TCP es stream y no message protocol?
10. ¿Por qué `recv` no corresponde a un `send`?
11. ¿Qué resuelve `sendall` y qué no?
12. ¿Qué diferencia packet de application message?
13. ¿Qué hace seguro un framing protocol?
14. ¿Qué diferencia latency, bandwidth, throughput y RTT?
15. ¿Qué problema resuelve DNS y qué no garantiza?
16. ¿Qué garantiza TCP y qué no?
17. ¿Por qué transport ACK no es application commit?
18. ¿Por qué flow control no sustituye backpressure?
19. ¿Cuándo UDP encaja y qué debe tolerar la app?
20. ¿Qué diferencia resolution, connect y read timeout?
21. ¿Por qué timeout produce ambiguous outcome?
22. ¿Cuándo retry duplica effects?
23. ¿Qué aporta una idempotency key y qué no?
24. ¿Qué diferencia HTTP status de transport success?
25. ¿Por qué 404 no significa unreachable?
26. ¿Por qué 500 no es connection error?
27. ¿Qué significa safe method?
28. ¿Qué hace idempotent un method?
29. ¿Por qué POST no se reintenta ciegamente?
30. ¿Por qué DELETE repetido puede responder distinto y seguir idempotent?
31. ¿Qué relaciona Content-Type con body?
32. ¿Qué separa JSON validity de domain validity?
33. ¿Por qué fragment no suele llegar al server?
34. ¿Por qué HTTP no equivale a una TCP nueva por request?
35. ¿Qué protege TLS y qué no valida?
36. ¿Cómo puede un firewall parecer timeout?
37. ¿Por qué ping no prueba disponibilidad del service?
38. ¿Cómo depuras “no conecta” por capas?
39. ¿Qué datos no deberían salir del host sin policy?
40. ¿Por qué remote response sigue siendo derived evidence?

## 32. Mini challenge — Protocolo local-first EIDOLON

### Objetivo y artefactos

```text
cs_m9_challenge/
├── framing.py
├── tcp_experiment.py
├── http_experiment.py
├── synthetic_records.jsonl
└── DIAGNOSTICS.md
```

Todo server usa loopback, port efímero, timeout y cleanup. Solo datos sintéticos.

### A. Raw TCP

1. Listener TCP en `127.0.0.1:0`.
2. Client conecta con timeout.
3. Protocolo length-prefix con maximum frame size.
4. Envía al menos dos frames y conserva boundaries.
5. `recv_exact` detecta EOF/truncamiento.
6. Todos los sockets/threads terminan bajo timeout.

### B. JSON message

7. Serializa synthetic record a JSON UTF-8.
8. Enmarca bytes y parsea en peer.
9. Valida `event_id`, `operation_id` y tipo mínimo después de parse.
10. Respuesta conserva operation ID; no prueba domain truth.

### C. Failure injection

11. Clasifica name resolution failure sin depender de Internet.
12. Simula connection refused sobre endpoint local controlado.
13. Fuerza read timeout con delayed server.
14. Trunca frame y espera `EOFError`.
15. Envía malformed JSON y espera error de parsing/application.
16. Cierra server antes de completar response.

### D. HTTP

17. `GET /health` retorna 200 + JSON.
18. `POST /operations` exige JSON/Content-Type y operation ID.
19. Missing path retorna 404; invalid body retorna 400; conflict retorna 409 según contrato.
20. Client observa status/headers/body por separado.

### E. Idempotency y ambiguous outcome

21. First POST commits y demora/pierde response.
22. Client agota timeout sin asumir failure definitivo.
23. Retry usa la misma `Idempotency-Key`.
24. Effect count permanece 1 y response indica duplicate/replay.
25. Documenta que memoria local no ofrece garantía tras restart.

### F. Diagnostics y privacy

26. `DIAGNOSTICS.md` traza DNS/host → endpoint/port → TCP → HTTP → JSON/domain.
27. Cada failure guarda operation ID, layer, exception/status y elapsed.
28. No guarda bodies, tokens, cookies ni conversaciones.
29. Distingue Python/OS/tool/platform-specific observations.

### G. Source discipline

30. `synthetic_records.jsonl` permanece byte por byte intacto.
31. Request/response/output se marca derived.
32. Remote-like response no sobrescribe source.

### Comprobaciones contractuales

**Continuación — adapta nombres:**

```python
source_before = source_path.read_bytes()

tcp_result = run_tcp_round_trip(records, timeout=1)
assert tcp_result == records

health = http_get_health(timeout=1)
assert health.status == 200
assert health.content_type == "application/json"

try:
    post_operation("op-001", timeout=0.02, inject_lost_response=True)
except TimeoutError:
    pass
else:
    raise AssertionError("ambiguous timeout not injected")

retry = post_operation("op-001", timeout=1)
assert retry.status == 200
assert retry.operation_id == "op-001"
assert effect_count("op-001") == 1

assert source_path.read_bytes() == source_before
assert all_servers_stopped()
```

### Failure cases obligatorios

- stream/message confusion corregida con framing;
- partial/truncated frame rechazado;
- malformed JSON separado de transport failure;
- refused/DNS/timeout/404 clasificados en capas distintas;
- timeout después de commit tratado como ambiguous;
- duplicate-safe retry con stable key;
- bind loopback verificado;
- body/secrets ausentes de logs;
- source intacto.

### Criterio de aprobación

- experiments funcionan offline con standard library;
- servers son loopback-only, acotados y cerrados;
- framing no depende de recv/send boundaries;
- TCP/UDP/HTTP guarantees no se mezclan;
- timeout/retry/idempotency tienen contrato explícito;
- status/JSON/domain failures son distinguibles;
- diagnostics avanzan por capas;
- privacy/source discipline son verificables;
- no aparecen FastAPI, auth, databases, WebSockets, proxies de producción, Docker ni AI.

## 33. Resumen

- Host, process, role y endpoint describen capas distintas.
- Endpoint TCP/UDP práctico incluye protocol, address y port.
- Loopback limita al host; localhost puede resolver IPv4/IPv6; `0.0.0.0` es wildcard de bind.
- Socket es un OS resource con lifecycle, no una pareja de números.
- Network transport mueve bytes; serialization/validation pertenecen a application.
- TCP es ordered reliable byte stream, no message protocol ni app acknowledgement.
- `recv`/`send` pueden ser parciales; `sendall` no demuestra remote commit.
- Framing define boundaries; packet no es application message.
- Latency, bandwidth, throughput y RTT no son intercambiables.
- DNS resolution/cache precede a connect y puede producir múltiples addresses.
- TCP flow control no limita por sí solo application backlog.
- UDP conserva datagrams sin reliability/order garantizados; se elige por requirements.
- Timeout identifica una frontera temporal, pero puede dejar outcome ambiguo.
- Retry seguro depende de effect semantics, budget y stable idempotency key.
- HTTP separa request/response semantics de transport success.
- Methods tienen safety/idempotency semántica; implementación debe respetarla.
- 404/500 son HTTP responses, no network unreachable.
- Content-Type guía interpretación; JSON válido no implica domain-valid.
- HTTP versions/connections varían; no asumas TCP nueva por request.
- TLS protege el canal bajo validación correcta, no la verdad del payload.
- Debugging identifica capa, recoge evidence mínima y asciende.
- EIDOLON conserva source; network response es derived evidence y egress es privacy boundary.

## 34. Checklist de dominio

- [ ] Puedo distinguir host, process, role y endpoint.
- [ ] Puedo explicar IPv4/IPv6/loopback/localhost.
- [ ] Puedo distinguir bind loopback y wildcard.
- [ ] Puedo explicar port y transport namespace.
- [ ] Puedo gestionar socket/listener/connection lifecycle.
- [ ] Puedo mostrar serialization → bytes → parsing.
- [ ] Puedo explicar TCP stream sin message boundaries.
- [ ] Puedo implementar bounded length-prefix framing.
- [ ] Puedo manejar partial read/EOF.
- [ ] Puedo explicar límites de sendall.
- [ ] Puedo distinguir packet y application message.
- [ ] Puedo diferenciar latency/bandwidth/throughput/RTT.
- [ ] Puedo explicar DNS/cache/multiple addresses.
- [ ] Puedo delimitar TCP guarantees.
- [ ] Puedo separar flow control y backpressure.
- [ ] Puedo elegir TCP/UDP por requirements.
- [ ] Puedo declarar resolution/connect/read/overall timeout.
- [ ] Puedo reconocer ambiguous outcome.
- [ ] Puedo justificar retry/idempotency key.
- [ ] Puedo interpretar HTTP request/response.
- [ ] Puedo descomponer URL y fragment.
- [ ] Puedo explicar method safety/idempotency.
- [ ] Puedo clasificar status codes por capa.
- [ ] Puedo validar Content-Type/JSON/domain por separado.
- [ ] Puedo explicar persistent/HTTP versions panorámicamente.
- [ ] Puedo explicar TLS y sus límites.
- [ ] Puedo ejecutar servers loopback con cleanup.
- [ ] Puedo depurar por DNS/TCP/TLS/HTTP/application.
- [ ] Puedo usar tools sin sobreinterpretarlas.
- [ ] Puedo redactar logs sin payload sensible.
- [ ] Puedo conservar source y clasificar response como derived.
- [ ] Puedo resolver el mini challenge con PF + CS-M1–CS-M9.

## 35. Preparación para CS-L17, CS-M10 y Backend Track

- **CS-L17 — Local request trace:** observa DNS/hosts, TCP connect y request HTTP local; clasifica cada fallo por capa.

| Concepto | Secciones | Evidencia | Lab/build |
|---|---:|---|---|
| Endpoint/loopback/socket | 1–4 | Guiados 1–4 | CS-L17 |
| Bytes/framing/TCP | 5–8, 11 | Guiados 5–7 | CS-L17 |
| DNS/failure layers | 10, 14, 23 | Guiados 9, 20 | CS-L17 |
| HTTP semantics | 16–20, 24 | Guiados 10–14 | CS-L17 |
| Timeout/retry/idempotency | 14–15, 25–26 | Guiados 15–18 | CS-L17, EIDOLON 0.0b |
| TLS/privacy/boundary | 21–23, 27 | Guiado 19 | CS-L17 |

Antes de avanzar entrega: local request trace, framed TCP exchange, timeout classification, 404-vs-network diagnosis, idempotent retry evidence, cleanup proof y redacted diagnostic log.

CS-M10 podrá relacionar network I/O con CPU/caches/dispositivos. Backend Track podrá concentrarse en API contracts/frameworks porque request, response, port, timeout, retry e idempotency ya tienen modelo mental. CS-M9 no implementa backend.

## 36. Recursos de ampliación

El módulo es autocontenido. Para ampliar usa [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) y documentación oficial de Python 3.14 para `socket`, `http.server`, `http.client`, `urllib` y `ssl`. Los estándares HTTP/TCP/TLS amplían garantías; no sustituyen los experiments.

## 37. Límite explícito del módulo

CS-M9 termina en layers, endpoints, IP/ports, sockets, bytes/framing, TCP/UDP, DNS, timeout/retry/idempotency, HTTP/TLS foundations, loopback exposure, diagnosis y privacy/source discipline.

No desarrolla subnetting/routing protocols, TCP algorithms formales, QUIC/TLS internals, packet crafting, FastAPI, advanced REST, auth/sessions/cookies/CORS/CSRF, WebSockets/SSE, proxies/load balancers, distributed systems, cloud networking, databases, Docker/Kubernetes ni real external APIs.

El siguiente paso permitido es revisar CS-M9 como `review candidate`. **No se crea ni se desarrolla CS-M10 en esta entrega.**
