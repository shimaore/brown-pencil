Serveur Vocal RIO

Deux middlewares:
- Middleware pour le trafic sortant. Intercepte les appels vers le 3179,
  utilise le numéro appelant, et donne les informations + notifie.
- Middleware pour le trafic entrant. Reçoit un appel, demande le numéro,
  vérifie qu'il existe et donne les informations + notifie.

Références: [Décision ARCEP 2013-0830](http://arcep.fr/uploads/tx_gsavis/13-0830.pdf), pages 46 et 47.
