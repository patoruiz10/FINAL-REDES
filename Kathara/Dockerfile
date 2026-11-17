# Imagen del router RA: Quagga (OSPF) + servidor DHCP
# Construir desde la raíz del repo: docker build -t kathara/quagga-dhcp Kathara/
FROM kathara/quagga
RUN apt-get update && apt-get install -y isc-dhcp-server && rm -rf /var/lib/apt/lists/*
