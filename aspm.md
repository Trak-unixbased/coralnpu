Sur Rockchip RK3588 : L'intégration PCIe est historiquement capricieuse sur ce SoC. Le contrôleur gère mal l'ASPM (Active State Power Management) et l'IOMMU lors des requêtes DMA du Coral. Pour éviter des kernel panics ou des timeouts du bus, il est souvent impératif de forcer pcie_aspm=off dans les paramètres de boot du noyau.


 La gestion de l'alimentation en état actif (ASPM, de l'anglais Active-State Power Management) permet de réaliser des économies d'énergie dans le sous-système interconnexion de composants périphériques (PCI Express, ou PCIe) en définissant un état d'alimentation plus faible pour les liens PCIe lorsque les périphériques auxquels ils sont connectés ne sont pas en cours d'utilisation. ASPM contrôle l'état de l'alimentation des deux côtés du lien, et permet d'économiser de l'énergie dans le lien même lorsque le périphérique se trouvant à la fin de celui-ci est dans un état d'alimentation complète.
Lorsque ASPM est activé, la latence de périphérique augmente à cause du temps requis pour effectuer la transition du lien entre les différents états d'alimentation. ASPM possède trois politiques afin de déterminer les états d'alimentation :

défaut
    établit les états d'alimentation du lien PCIe en fonction des valeurs par défaut spécifiées par le microprogramme du système (par exemple, BIOS). Ceci est l'état par défaut d'ASPM. 
powersave
    règle ASPM de manière à économiser de l'énergie à chaque occasion, sans tenir compte de la performance en résultant. 
performance
    désactive ASPM afin de permettre aux liens PCIe d'opérer avec une performance maximale. 

La prise en charge ASPM peut être activée ou désactivée par le paramètre du noyau pcie_aspm, dans lequel pcie_aspm=off désactive ASPM et pcie_aspm=force active ASPM, même sur les périphériques ne prenant pas en charge ASPM.
Les stratégies ASPM sont définies dans /sys/module/pcie_aspm/parameters/policy, mais peuvent aussi être spécifiées lors du démarrage avec le paramètre de noyau pcie_aspm.policy, dans lequel pcie_aspm.policy=performance définira par exemple la stratégie de performance ASPM. 
