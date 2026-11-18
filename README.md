# Final – Redes y Comunicación

Diseño y configuración de una red con enrutamiento dinámico (OSPF) en Kathará.

- **Alumno:** Patricio Ruiz
- **Profesor:** Ing. Ricardo Brisighelli
- **Institución:** Instituto Zona Oeste N.º 9045 – 3er Año 2026


## Estructura del repositorio

```
FINAL-REDES/
├── Documentación/
│   ├── Esquema Conectividad/   Esquema de la red (JPG, PDF y editable draw.io)
│   ├── Dijkstra/               Árbol de expansión desde RA (tabla y esquema)
│   └── Direccionamiento IP/    Cálculo y tablas de direccionamiento (DOCX y PDF)
└── Kathara/                    Laboratorio
    ├── lab.conf
    ├── Dockerfile              Imagen del router RA (Quagga + DHCP)
    ├── *.startup
    └── <equipo>/etc/...        Configuración de cada equipo
```


## Cómo levantar el laboratorio

### 1. Construir la imagen del router RA

RA es el router conectado a Internet y, además de OSPF, corre el servidor DHCP de LA.
Ninguna imagen oficial de Kathará trae Quagga y el servidor DHCP juntos, por eso se usa una imagen propia basada en `kathara/quagga`.
El resto de los equipos usa imágenes oficiales, indicadas en `lab.conf` (`kathara/quagga` en los routers, `kathara/bind` en HA1 y `kathara/base` en HB1).

Desde la raíz del repositorio:

```
docker build -t kathara/quagga-dhcp Kathara/
```

Se hace una sola vez por equipo. Para verificarla:

```
docker run --rm --entrypoint sh kathara/quagga-dhcp -c "which dhcpd zebra ospfd iptables"
```

Tienen que aparecer las cuatro rutas.

### 2. API de login

El servidor HTTP de HB1 reenvía los pedidos a la API de login, que corre fuera del laboratorio en `192.168.1.8:3000`.
Tiene que estar en ejecución y accesible desde el anfitrión; si no, Apache responde `502 Bad Gateway`.

Con eso, el laboratorio se levanta normalmente con `kathara lstart` desde la carpeta `Kathara/`.


## Servicios configurados

| Servicio | Equipo | Detalle |
|---|---|---|
| OSPF | 20 routers (Quagga) | Área 0, costos según las métricas del esquema |
| Salida a Internet | RA | Interfaz bridged (eth3) con NAT y ruta por defecto anunciada por OSPF |
| DNS | HA1 (`ha`, 10.0.0.1) | Zona `selvent.izo`, resolución inversa y forwarders a 8.8.8.8 / 8.8.4.4 |
| DHCP | RA (LA) | HA1 y HA2 con IP fija por MAC; HA3, HA4 y HA5 con IP dinámica (10.0.0.10–10.0.0.100); concesión de 5 minutos |
| HTTP | HB1 (`hb`, 172.16.0.1) | Apache como proxy inverso hacia la API de login (192.168.1.8:3000), publicado hacia Internet con DNAT en RA (puerto 80) |

## Verificación rápida

```
kathara connect ra
vtysh -c "show ip ospf neighbor"     # vecinos en estado Full
vtysh -c "show ip route ospf"        # rutas aprendidas por OSPF

kathara connect hl
ping -c 3 ha.selvent.izo             # conectividad atravesando la red
dig -x 192.168.0.1                   # resolución inversa
ping -c 3 google.com                 # salida a Internet
```

Servidor HTTP desde Internet (en el equipo anfitrión, fuera del laboratorio):

```
curl -v -X POST http://172.17.0.2/inicioSesion -u <usuario>:<clave>
```

`172.17.0.2` es la IP de RA en la red de Docker (interfaz de Internet). Si Docker le asignó otra, se ve con `ip a show eth3` en RA.
