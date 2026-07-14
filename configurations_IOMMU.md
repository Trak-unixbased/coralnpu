Quels sont les paramètres de boot et les configurations IOMMU nécessaires pour stabiliser le PCIe du RK3588 avec un Coral TPU ? 

Le couplage d'un SoC Rockchip RK3588 avec un accélérateur Coral Edge TPU est notoirement instable "en sortie de boîte". Les plantages (souvent des *kernel panics* ou des *timeouts* du bus PCIe) proviennent de deux facteurs : une gestion agressive et défaillante de l'économie d'énergie (ASPM) par le contrôleur PCIe du Rockchip, et des conflits de traduction d'adresses mémoire (DMA) avec le SMMU (l'IOMMU d'ARM).

Le problème est particulièrement marqué sur les modules Dual Edge TPU, car ces derniers embarquent un switch PCIe intermédiaire (souvent Asmedia) qui digère très mal les transitions d'état d'alimentation imposées par le noyau.

Voici la configuration bas niveau nécessaire pour stabiliser l'ensemble.

## 1. Paramètres de boot (Kernel Cmdline)

Il faut éditer la ligne de commande du noyau (généralement dans `/boot/extlinux/extlinux.conf` ou via U-Boot) pour passer les paramètres suivants :

* **`pcie_aspm=off`**
C'est le paramètre le plus critique. L'Active State Power Management (ASPM) tente de basculer le lien PCIe dans des états de basse consommation (L0s, L1) lors des périodes d'inactivité. Sur le RK3588, la sortie de veille du bus est souvent trop lente ou échoue, ce qui désynchronise le module Coral de manière irrécupérable. Désactiver l'ASPM maintient le lien PCIe actif en permanence.
* **`iommu=pt`** (ou `iommu.passthrough=1` selon les patchs du noyau)
Ce paramètre force l'IOMMU en mode *passthrough*. Le driver `apex` du Coral alloue de gros blocs de mémoire contiguë pour les transferts DMA. Si le SMMU du Rockchip reste en mode strict, il intercepte et bloque souvent ces requêtes, générant des erreurs de type `DMA translation fault` dans `dmesg`. Le mode *passthrough* permet aux périphériques de confiance d'accéder directement à la mémoire physique.
* **`swiotlb=65536`** (Optionnel mais recommandé)
Le RK3588 gérant souvent plus de 4 Go de RAM, le noyau utilise des *bounce buffers* (SWIOTLB) pour les périphériques limités à un adressage 32 bits. Si la zone allouée par défaut est trop petite lors d'inférences lourdes, le système plantera. Cette valeur étend le buffer à 128 Mo.

## 2. Configuration du Device Tree (DTS) - En cas d'échec

Si le TPU décroche toujours malgré les paramètres de boot (erreur `link down` dans les logs), le problème vient de l'intégrité du signal matériel (PHY).

L'implémentation du PCIe 3.0 (Gen3) sur de nombreuses *carrier boards* RK3588 souffre d'interférences. Le Coral TPU est très sensible à la gigue d'horloge (*clock jitter*). La solution radicale consiste à dégrader la vitesse du port au niveau du Device Tree.

Dans le fichier DTS de la carte, localisez le nœud PCIe correspondant (par exemple `pcie30pb`) et modifiez ou ajoutez cette ligne :

```dts
&pcie30pb {
    status = "okay";
    max-link-speed = <2>; /* Force le lien en PCIe Gen2 (5 GT/s) au lieu de Gen3 */
    num-lanes = <4>;
};

```

La bande passante du PCIe Gen2 (environ 2 Go/s sur 4 lignes) est largement suffisante pour ne pas brider les transferts de tenseurs vers le Coral, et la stabilité du lien s'en trouvera grandement améliorée.

## 3. Validation post-reboot

Après redémarrage, vérifiez la bonne prise en compte des paramètres avec ces commandes :

1. **Vérifier l'ASPM :**
`dmesg | grep -i aspm` (doit indiquer que l'ASPM est désactivé)
2. **Vérifier le mode Passthrough :**
`dmesg | grep -i iommu` (recherchez une mention du mode "passthrough" ou "bypass")
3. **Vérifier le chargement du driver :**
`lspci -nnkd 1ac1:089a` (doit lister le TPU avec le kernel driver `apex` en cours d'utilisation)
