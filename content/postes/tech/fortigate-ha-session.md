# Gestion des sessions Haute Disponibilité sur FortiGate

Article concernant un retour d'expérience sur FortiGate

Lors d'une bascule du membre actif vers le passif dans un cluster, nous avons observé un bagot de 5 minutes que nous n'avions pas habituellement.

Après analyse de la configuration, en suivant les recommendations de Fortinet, l'intégrateur avait deux lignes de commandes.

Seulement, les ports HA

Dans notre outil de Monitoring;

Nous devions réaliser une nouvelle bascule alors.

Parfois le mieux est l'ennemi du bien.