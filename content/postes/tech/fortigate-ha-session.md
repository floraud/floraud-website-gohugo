+++
title = 'Gestion des sessions en haute disponibilité sur FortiGate'
categories = ["Tech"]
tags = ["Reseau", "Fortinet", "FortiGate", "Firewall", "Troubleshooting"]
featured_image = "/images/tech/fortigate-ha-session.webp"
date = 2026-04-23T18:57:11+02:00
+++

Cet article concerne un retour d'expérience sur FortiGate où le session-pickup ne s'est pas comporté comme prévu.

<!--more-->

Lors d'une bascule du membre actif vers le passif dans un cluster, nous avons observé une instabilité de 5 minutes que nous n'avions pas auparavant.

Habituellement, pour que la bascule se déroule de manière transparente, nous aimons employer la commande :

```
config system ha
  set session-pickup enable
end
```

Après analyse de la configuration, nous avons identifié que l'intégrateur avait ajouté cette ligne de commande pour éviter que le session-pickup ne soit trop consommateur sur la partie haute disponibilité : `set session-pickup-delay enable`.

En se basant sur la [documentation officielle](https://docs.fortinet.com/document/fortigate/6.4.0/ports-and-protocols/796662/fgsp-fortigate-session-life-support-protocol), on apprend que cela permet de ne reprendre que les flux actifs dont les sessions durent plus de 30 secondes. Mais le ressenti utilisateur comme la supervision ont montré que les connexions ont été instables pour une durée de 5 minutes.

Devant rebasculer dans l'autre sens et par souci homogénéisation, j'ai pris la décision de retirer la commande en question. Ainsi, la bascule suivante s'est déroulée sans impact. Je ne recommande donc pas de passer cette commande par anticipation mais seulement si un jour vous avez une bascule qui se déroule mal.

*Photo de bannière par [Waldemar Brandt](https://unsplash.com/@waldemarbrandt67w?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) sur [Unsplash](https://unsplash.com/photos/brown-brick-wall-rhaS97NhnHg?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*