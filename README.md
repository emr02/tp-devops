# TP DevOps Correction Docker

Correction de la partie Docker du module DevOps. Amusez-vous bien avec GitHub Actions !

Additional processes

1. MR d'une branche à la branche develop :
a. disable le merge si le build ou les tests fail
b. pas de push sur le docker hub

2. MR de devlop a main :
a. disable le merge si le build ou les tests fail
b. Image avec le tag de version incrementé
c. Push sur le docker Hub
