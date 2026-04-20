# Guia de Implementação - Projeto RC2 (2025/2026)

**Grupo:** 14060 e 27131
**Rede Base ($10.F.0.0/16$):** 10.236.0.0/16 (F = 14060 % 256 = 236)

---

## 1. Planeamento de Endereçamento (VLSM)

As sub-redes foram calculadas da maior para a menor necessidade para maximizar a eficiência.

| Sub-rede | Necessidade | Hosts Reais | Endereço de Rede | Máscara (CIDR) | Gateway (Exemplo) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 30** | 1023 | 1024 | 10.236.0.0 | /22 | 10.236.0.1 |
| **Subrede D** | 500 | 512 | 10.236.4.0 | /23 | 10.236.4.1 |
| **VLAN 20** | 200 | 256 | 10.236.6.0 | /24 | 10.236.6.1 |
| **Subrede A** | 20 | 32 | 10.236.7.0 | /27 | 10.236.7.1 |
| **VLAN 10** | 6 | 8 | 10.236.7.32 | /29 | 10.236.7.33 |
| **Subrede B**| 2 | 4 | 10.236.7.40 | /30 | 10.236.7.41 |
| **Subrede C**| 2 | 4 | 10.236.7.44 | /30 | 10.236.7.45 |

---

## 2. Loopbacks (REQ 3)
IPs para gestão/estabilidade OSPF.

- **R1:** 1.1.1.1/32
- **R2:** 2.2.2.2/32
- **R3:** 3.3.3.3/32

**Mikrotik (R1):** `/interface loopback add name=lo0; /ip address add address=1.1.1.1 interface=lo0`
**Cisco (R2/R3):** `int lo0; ip address X.X.X.X 255.255.255.255`


---

## 2. Configurações Base (Router 1 - Mikrotik)

O Router 1 (R1) liga à Internet e gere os serviços de DHCP e Firewall avançada.

### A. Interfaces e WAN
```bash
# Configurar WAN (DHCP Client)
/ip dhcp-client add interface=ether1 disabled=no

# Configurar IPs Estáticos (Exemplos)
/ip address add address=10.236.7.1/27 interface=ether2 comment="Subrede A"
/ip address add address=10.236.7.41/30 interface=ether3 comment="Subrede B"

### D. Config Série (REQ 2)
Se R1 = DCE na Subrede C:
`/interface pppoe-client` (ou config específica de driver GNS3 Serial).
No Cisco R2/R3: `clock rate 64000` (se DCE).

```

### B. DHCP Server (para PCs e VM)
```bash
/ip pool add name=pool_subredeD ranges=10.236.4.10-10.236.5.250
/ip dhcp-server add address-pool=pool_subredeD interface=ether4 name=dhcp_D disabled=no
/ip dhcp-server network add address=10.236.4.0/23 gateway=10.236.4.1 dns-server=8.8.8.8
```

### C. NAT (Acesso à Internet)
```bash
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

---

## 3. Encaminhamento OSPF (Toda a Topologia)

Configuração para permitir que todos os routers conheçam as rotas da rede.

**No Mikrotik (R1):**
```bash
/routing ospf instance add name=default router-id=1.1.1.1
/routing ospf area add instance=default name=area0 area-id=0.0.0.0
/routing ospf network add network=10.236.0.0/16 area=area0
```

**Nos Cisco (R2 e R3):**
```bash
router ospf 1
 router-id 2.2.2.2 (ou 3.3.3.3 no R3)
 network 10.236.0.0 0.0.255.255 area 0
```

---

## 4. Segurança e Firewall

### A. Port Knocking (R1 - Mikrotik)
Permite login apenas após bater nas portas 3000, 4000 e 5000 (TCP).

```bash
/ip firewall filter
add chain=input protocol=tcp dst-port=3000 action=add-src-to-address-list address-list=knock1 address-list-timeout=15s
add chain=input protocol=tcp dst-port=4000 address-list=knock1 action=add-src-to-address-list address-list=knock2 address-list-timeout=15s
add chain=input protocol=tcp dst-port=5000 address-list=knock2 action=add-src-to-address-list address-list=access_granted address-list-timeout=30m
add chain=input protocol=tcp dst-port=8291 src-address-list=!access_granted action=drop comment="Bloquear Winbox sem knock"
```

### B. Restrições da VM (Horários e Sites)
Apenas Finanças e Seg. Social em horários específicos.

```bash
# 1. Criar lista de sites permitidos (resolvidos por DNS)
/ip firewall address-list add address=www.portaldasfinancas.gov.pt list=sites_permitidos
/ip firewall address-list add address=www.seg-social.pt list=sites_permitidos

# 2. Bloquear tudo exceto sites permitidos nos horários definidos
/ip firewall filter
add chain=forward src-address=10.236.4.x (IP da VM) dst-address-list=sites_permitidos time=09:00:00-12:00:00,mon,tue,wed,thu,fri action=accept
add chain=forward src-address=10.236.4.x (IP da VM) dst-address-list=sites_permitidos time=14:00:00-17:00:00,mon,tue,wed,thu,fri action=accept
add chain=forward src-address=10.236.4.x (IP da VM) action=drop

### D. Bloqueio Inter-PC (REQ 6)
PC2, PC3 e PC4 isolados.
**Mikrotik (R1 - se PCs na mesma bridge/switch):**
`/interface bridge filter add chain=forward src-mac-address=... dst-mac-address=... action=drop`
**OU via IP Firewall (R1/R3):**
`/ip firewall filter add chain=forward src-address=10.236.x.x (PC2) dst-address=10.236.x.x (PC3) action=drop` (repetir para todos pares).

```

### C. Limite de PC2 (Bandwidth)
Velocidade de 1 Mbit e acesso entre as 17h e 23h.

```bash
/queue simple add name=limite_pc2 target=10.236.x.x (IP do PC2) max-limit=1M/1M time=17:00:00-23:00:00,mon,tue,wed,thu,fri
```

---

## 5. ACLs Cisco (R2 & R3)

### A. R3 Inter-VLAN (REQ 11)
Bloquear tudo entre VLANs exceto 80(HTTP) e 443(HTTPS).
```bash
ip access-list extended VLAN_FILTER
 permit tcp 10.236.0.0 0.0.15.255 10.236.0.0 0.0.15.255 eq 80
 permit tcp 10.236.0.0 0.0.15.255 10.236.0.0 0.0.15.255 eq 443
 deny ip 10.236.0.0 0.0.15.255 10.236.0.0 0.0.15.255
 permit ip any any
!
interface FastEthernet0/0.10 (e outras sub-interfaces)
 ip access-group VLAN_FILTER in
```

### B. Telnet R2 (REQ 7)


```bash
access-list 10 permit 10.236.7.x (IP do PC1)
line vty 0 4
 password rc2
 login
 access-class 10 in
exit
enable password rc2
```

---

## 6. API Mikrotik (Python)
Exemplo de script para ler nome, IPs e alterar password.

```python
import routeros_api

connection = routeros_api.RouterOsApiPool('10.236.7.1', username='admin', password='old_password')
api = connection.get_api()

# 1. Visualizar Nome e IPs
identity = api.get_resource('/system/identity').get()
print(f"Router Name: {identity[0]['name']}")

addresses = api.get_resource('/ip/address').get()
for addr in addresses:
    print(f"Interface: {addr['interface']} - IP: {addr['address']}")

# 2. Alterar Password
# Nota: Requer lógica adicional para validar a password antiga antes de enviar o comando 'set'
users = api.get_resource('/user')
users.set(id='admin', password='new_password')

connection.disconnect()
```

---

## 7. Notas Finais para o Relatório
- **Qualidade do Relatório:** Incluir prints das tabelas de rotas (`ip route print` no Mikrotik / `show ip route` no Cisco).
- **Subrede C:** Certificar que as interfaces estão configuradas como `encapsulation ppp` se usar interfaces de série reais.
