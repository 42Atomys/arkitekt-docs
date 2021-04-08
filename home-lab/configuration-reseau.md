---
description: >-
  Cette page regroupe toutes les configuration du réseau. Routeur, Switch,
  Firewall
---

# Configuration réseau

Sur la partie software, j'ai garder les firewares de base du router, firewall et switch donc rien de spécial. 

{% hint style="info" %}
Les trois serveurs tournent sous Debian 10.
{% endhint %}

## Routeur

Le router a été configurer avec son firewall comme première porte du réseau. Son interface `eth0` connecté a la box à un adressage statique en `192.168.1.200/24`, le duplex et la speed du port sont en autonégociation.

Le port eth8 lui est configurer comme étant le router du sous-réseau avec une adresse statique `172.28.0.1/24`, le duplex et la speed du port sont eux aussi en autonégociation.

{% hint style="warning" %}
Une tentative de support de l'ipv6 à été faite en mettant en place du prefix-delegation mais malheureusement la Livebox ne fourni qu'un /56 et le prefix-delegation n'est pas possible en DMZ. J'ai dont une ipv6 sur l'eth0 en autoconf mais pas de propagation au réseau suivant.
{% endhint %}

```javascript
ethernet eth0 {
  address 192.168.1.200/24
  address fe80::0:1/64
  duplex auto
  firewall {
    in {
      name WAN_IN
    }
    local {
      name WAN_LOCAL
    }
  }
  ipv6 {
    address {
      autoconf
    }
  }
  speed auto
}
ethernet eth8 {
  address 172.28.0.1/24
  address fe80::8:1/64
  duplex auto
  speed auto
}
```

> Voici la configuration des interfaces sous forme plus exhaustive

Le router dispose d'une firewall intégré qui est en drop par défaut et qui sert de firewall depuis l'extérieur.

```javascript
all-ping enable
broadcast-ping disable
name WAN_IN {
  default-action drop
  description "WAN to internal"
  rule 10 {
    action accept
    description "Allow established/related"
    protocol all
    state {
      established enable
      invalid disable
      new disable
      related enable
    }
  }
  rule 20 {
    action accept
    description ICMP
    protocol icmp
    state {
      established enable
      invalid enable
      new enable
      related enable
    }
  }
  rule 30 {
    action accept
    description "Allow WAN Local SSH"
    destination {
      port 4222
    }
    protocol tcp
    source {
      address 192.168.0.0/16
    }
  }
  rule 40 {
    action drop
    description "Drop invalid state"
    log disable
    protocol all
    state {
      established disable
      invalid enable
      new disable
      related disable
    }
  }
}
name WAN_LOCAL {
  default-action drop
  description "WAN to router"
  rule 10 {
    action accept
    description "Allow established/related"
    protocol all
  }
  rule 20 {
    action drop
    description "Drop invalid state"
    protocol all
  }
}
receive-redirects disable
send-redirects enable
source-validation disable
syn-cookies enable
```

> Voici la configuration du firewall du router

Afin de palier a des soucis lors du debug du réseau, et aussi par flemme, le port eth8 dispose d'un serveur DHCP.

```javascript
dhcp-server {
  hostfile-update disable
  shared-network-name dhcp-8 {
    authoritative disable
    subnet 172.28.0.0/24 {
      default-router 172.28.0.1
      dns-server 8.8.8.8
      dns-server 8.8.4.4
      lease 86400
      start 172.28.0.10 {
        stop 172.28.0.100
      }
    }
  }
  static-arp disable 
  use-dnsmasq disable
}
```

> Voici la configuration du serveur DHCP sur le port ETH8

Si des services doivent être contacté publiquement, c'est sur le router qu'il faudra configurer en mode `port-forward` vers l'IP de destination où le serveur qui fourni le service. Dans notre cas cela sera une VM avec KVM et Qemu.  
La configuration du réseau est spéciale de manière à pouvoir directement contacter les VMs sans passer par un bridge en mode Gateway. Pour en savoir plus cette configuration est expliqué sur la page suivante

{% page-ref page="configuration-reseau-kvm.md" %}

#### Example  de port forward

Si nous avons un service web \(Nginx\) sur la VM `172.16.1.3` sur le port 80, il faudra ajouter la route statique pour le réseau du firewall ainsi que définir son `port-forward`, définition ci-dessous:

```javascript
// Route definition, next-hop is the firewall
route 172.16.0.0/16 {
  next-hop 172.28.0.2 {
  }
}
```

```javascript
// Rule definition to forward 80 port to VM (KVm based) on firewall LAN1 network
rule 1 {
  description "Nginx Forward to Gateway"
  forward-to {
    address 172.16.1.3
  }
  original-port 80
  protocol tcp
}
```

## Firewall

Le firewall est configuré via l'interface **UnifyNetworkController**, par défaut tout le réseau est bloqué. Il accepte le réseau du router via l'interface WAN1 et est en configuration manuelle pour avoir une IP en 172.28.0.2/16 et en Gateway `172.28.0.1/16` qui est le router Infinity précédemment configuré.

La configuration du LAN1 \(ou est connecté le switch et les serveurs\) est la suivante:

| Setting | Value |
| :--- | :--- |
| Gateway IP | 172.16.0.1 |
| Network Broadcast IP | 172.16.255.255 |
| Network IP Count | 65,534 |
| Network IP Range | 172.16.0.1 - 172.16.255.254 |
| Network Subnet Mask | 255.255.0.0 |
| DHCP Range | 172.16.244.46 - 172.16.244.254 |
| DHCP DNS Servers | 172.16.1.2 - 8.8.8.8 - 8.8.4.4 |
| DHCP Lease Time | 86400 |
| DHCP Domain Name | atomys.lab |

C'est cette Gateway qui va avoir accès a tout nos serveurs physiques ainsi qu'aux VMs présente sur les serveurs du réseaux.

{% hint style="info" %}
**NOTE**: Pensez à ajouter vos règles de firewall une fois le network configurer !
{% endhint %}

## Switch

Le switch est en auto-configuration, nous avons juste attribué une IP statique `172.16.0.2/16`

