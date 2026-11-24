# LAB – DHCP, NAT Estático, PAT, Roteamento Estático, VLAN e SVI

## 1. Objetivo do Laboratório

Este laboratório tem como objetivo demonstrar, na prática, como implementar e configurar os principais recursos de redes em dispositivos **Cisco**, abordando os seguintes conceitos:

- DHCP (Dynamic Host Configuration Protocol);
- NAT Estático (Network Address Translation);
- PAT (Port Address Translation);
- Roteamento Estático;
- VLAN (Virtual Local Area Network);
- SVI (Switched Virtual Interface).

Durante o desenvolvimento do laboratório, serão realizadas configurações de endereçamento IP, segmentação de redes, distribuição automática de endereços IP, roteamento entre redes e tradução de endereços para acesso à internet.

---

## 2. Topologia e Configuração de Endereçamento IP

### R1 – Roteador Cisco

| Interface | Endereço IP | Descrição |
|---|---|---|
| Fa0/0 | DHCP Client | Interface de saída para internet |
| Fa1/0 | 10.0.0.1/30 | Ponto a ponto (PTP) com SW |
| Fa2/0 | 172.16.0.1/24 | Gateway para Windows Server |

### SW – Switch Cisco

| Interface | Endereço IP | Descrição |
|---|---|---|
| E0/0 | 10.0.0.2/30 | Ponto a ponto (PTP) com R1 |
| VLAN 10 | 192.168.10.1/24 | SVI e Gateway da VLAN 10 |
| VLAN 20 | 192.168.20.1/24 | SVI e Gateway da VLAN 20 |

### Windows Server

| Interface | Endereço IP | Configuração |
|---|---|---|
| E0 | 172.16.0.100/24 | IP Estático |
| Gateway | 172.16.0.1 | Gateway padrão |

---

## 3. Configuração das VLANs no Switch

### 3.1. Criando as VLANs

```cisco
enable
configure terminal

vlan 10
 name MARKETING

vlan 20
 name RH

end
```

### 3.2. Associando as portas às VLANs

**VLAN 10 – MARKETING**

```cisco
enable
configure terminal

interface range e0/1-2
 switchport mode access
 switchport access vlan 10

end
```

**VLAN 20 – RH**

```cisco
enable
configure terminal

interface range e0/3, e1/0
 switchport mode access
 switchport access vlan 20

end
```

---

## 4. Configuração das SVI (Switched Virtual Interface)

As interfaces SVI permitem configurar endereços IP diretamente nas VLANs, possibilitando a comunicação entre diferentes redes por meio do roteamento de camada 3.

**Configurando a SVI da VLAN 10:**

```cisco
enable
configure terminal

interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

exit
```

**Configurando a SVI da VLAN 20:**

```cisco
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

end
```

---

## 5. Configuração do DHCP

O DHCP será responsável por distribuir automaticamente os endereços IP aos dispositivos conectados às VLANs 10 e 20.

### 5.1. Reservando endereços IP

Os endereços reservados não serão distribuídos pelo servidor DHCP, permitindo sua utilização em dispositivos com endereçamento estático.

```cisco
enable
configure terminal

ip dhcp excluded-address 192.168.10.1 192.168.10.50
ip dhcp excluded-address 192.168.20.1 192.168.20.50

end
```

### 5.2. Configurando o DHCP – VLAN 10

```cisco
enable
configure terminal

service dhcp

ip dhcp pool LAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8

end
```

### 5.3. Configurando o DHCP – VLAN 20

```cisco
enable
configure terminal

service dhcp

ip dhcp pool LAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8

end
```

---

## 6. Configuração do Roteamento Estático

O roteamento estático será utilizado para permitir a comunicação entre as redes conectadas ao roteador R1 e ao switch SW, além de definir a rota padrão para a saída à internet.

### 6.1. Configuração no R1

```cisco
ip route 0.0.0.0 0.0.0.0 192.168.14.2

ip route 192.168.10.0 255.255.255.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 10.0.0.2

ip default-gateway 192.168.14.2
```

### 6.2. Configuração no SW

```cisco
ip route 0.0.0.0 0.0.0.0 10.0.0.1

ip route 192.168.14.0 255.255.255.0 10.0.0.1
```

---

## 7. Configuração do NAT e PAT

O NAT e o PAT serão configurados no roteador R1 para permitir que os dispositivos da rede interna utilizem a interface externa para comunicação com outras redes.

### 7.1. Definindo as interfaces NAT

**Interface Outside:**

A interface FastEthernet0/0 será definida como interface externa, responsável pela comunicação com a rede de saída.

```cisco
interface FastEthernet0/0
 ip nat outside
```

**Interfaces Inside:**

As interfaces FastEthernet1/0 e FastEthernet2/0 serão definidas como interfaces internas.

```cisco
interface FastEthernet1/0
 ip nat inside

interface FastEthernet2/0
 ip nat inside
```

### 7.2. Criando a ACL para o PAT

A ACL será responsável por identificar quais redes internas poderão utilizar o PAT.

```cisco
ip access-list standard LAN
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
```

### 7.3. Configurando o NAT Estático

O NAT Estático permite realizar o mapeamento fixo de um endereço IP interno para um endereço externo.

```cisco
ip nat inside source static 172.16.0.100 interface FastEthernet0/0
```

### 7.4. Configurando o PAT (NAT Overload)

O PAT permite que vários dispositivos da rede interna compartilhem o endereço IP da interface externa, diferenciando as conexões por meio das portas de origem.

```cisco
ip nat inside source list LAN interface FastEthernet0/0 overload
```

---
Adicione o tópico abaixo ao final do seu documento, após as considerações finais ou antes delas, para validar o funcionamento das configurações realizadas no laboratório.

---

## 8. Testes e Verificação de Conectividade

Após concluir as configurações de VLAN, SVI, DHCP, roteamento estático e NAT/PAT, é necessário realizar testes para validar o funcionamento da topologia e identificar possíveis problemas de conectividade.

### 8.1. Verificando as VLANs

No switch Cisco, utilize os comandos abaixo para verificar se as VLANs foram criadas corretamente e se as portas estão associadas às VLANs correspondentes.

```
show vlan brief
```

### 8.2. Verificando as interfaces SVI

Verifique se as interfaces VLAN estão ativas e se os endereços IP foram configurados corretamente.

```
show ip interface brief
```

Para consultar os detalhes de uma interface específica:

```
show interfaces vlan 10
show interfaces vlan 20
```

### 8.3. Verificando o DHCP

No dispositivo responsável pelo serviço DHCP, execute os comandos abaixo para verificar os pools configurados e os endereços IP distribuídos.

```
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
```

### 8.4. Verificando o roteamento

No roteador R1 e no switch Layer 3, utilize os comandos abaixo para consultar a tabela de roteamento e verificar as rotas configuradas.

```
show ip route
show running-config | include ip route
```

### 8.5. Verificando o NAT e PAT

No roteador R1, execute os seguintes comandos para verificar as configurações e traduções NAT.

**Consultar as traduções NAT:**

```
show ip nat translations
```

**Consultar estatísticas do NAT:**

```
show ip nat statistics
```

**Verificar as interfaces Inside e Outside:**

```
show running-config | include ip nat
```

**Verificar a ACL utilizada pelo PAT:**

```
show access-lists LAN
```

### 8.6. Testando a conectividade entre dispositivos

Realize testes de conectividade utilizando o comando `ping` para verificar a comunicação entre os dispositivos e os gateways das respectivas redes.

**Testando o gateway da VLAN 10:**

```
ping 192.168.10.1
```

**Testando o gateway da VLAN 20:**

```
ping 192.168.20.1
```

**Testando a comunicação com o R1:**

```
ping 10.0.0.1
```

**Testando a comunicação com o Windows Server:**

```
ping 172.16.0.100
```

### 8.7. Testando o acesso à internet

Para validar o funcionamento do roteamento e do NAT/PAT, execute um teste de conectividade com um endereço IP público.

**Teste de conectividade com o DNS público do Google:**

```
ping 8.8.8.8
```

Se houver resposta, significa que existe conectividade IP com o destino externo.

Para validar o funcionamento da resolução DNS, também é possível executar:

```
ping google.com
```

### 8.8. Validando as traduções NAT durante os testes

Enquanto os dispositivos das VLANs realizam o ping para `8.8.8.8`, execute no R1:

```
show ip nat translations
```

O objetivo é verificar se os endereços IP privados das VLANs estão sendo traduzidos para o endereço IP da interface externa do roteador.

Também é possível acompanhar os contadores de tradução utilizando:

```
show ip nat statistics
```

### 8.9. Resultado esperado

Ao finalizar os testes, espera-se que:

- As VLANs 10 e 20 estejam criadas e com suas respectivas portas associadas.
- As interfaces SVI estejam operacionais.
- Os dispositivos recebam endereços IP automaticamente por DHCP.
- Exista comunicação entre as VLANs, quando o switch Layer 3 estiver realizando o roteamento.
- O roteamento entre o switch e o R1 esteja funcionando.
- O NAT Estático realize a tradução do endereço configurado.
- O PAT permita que múltiplos dispositivos compartilhem o endereço IP externo.
- Os dispositivos das redes internas consigam alcançar o endereço `8.8.8.8`.
- As traduções sejam visualizadas corretamente na tabela NAT do R1.

**Dica para o laboratório:** execute o `show ip nat translations` antes e depois de realizar o ping para `8.8.8.8`. Assim, você consegue demonstrar visualmente no vídeo como o PAT cria as traduções dos endereços privados para o endereço IP externo do roteador.

## 9. Considerações Finais

Ao concluir este laboratório, você terá praticado configurações fundamentais para a administração de redes Cisco, incluindo segmentação de redes com VLANs, configuração de gateways utilizando SVI, distribuição de endereços IP por DHCP, roteamento estático e implementação de NAT/PAT.

Esses conceitos são fundamentais para compreender o funcionamento de redes corporativas e desenvolver habilidades práticas de configuração e troubleshooting em ambientes de infraestrutura de redes.
