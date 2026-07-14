Le dépôt spécifique `Trak-unixbased/coralnpu` n'étant pas indexé publiquement à cet instant, l'évaluation se base sur les obstacles structurels incontournables qu'un fork doit résoudre pour compiler la stack Google Coral Edge TPU sur cette configuration.

La compilation pour **aarch64 sur Ubuntu 26.04** est **techniquement réalisable, mais complexe**. Le matériel ne sera pas le point de blocage principal ; le défi réside dans la transition vers les noyaux 7.x et les changements drastiques de l'API C dans Python 3.14.

## 1. Couche Kernel (Drivers Gasket / Apex)

Pour que le Coral (version M.2/PCIe) soit détecté, le système a besoin des modules hors-arbre `gasket` et `apex`. Le code original de Google ne compile plus depuis les noyaux 5.15.

* **Sur CIX CD8180 (Armv9) :** Cette puce récente dispose d'un contrôleur PCIe Gen4 natif. La stabilité matérielle devrait être au rendez-vous. L'enjeu sera purement logiciel : le fork doit inclure un `gasket-dkms` patché pour réussir la compilation contre les *headers* du noyau 7.x embarqué par Ubuntu 26.04.
* **Sur Rockchip RK3588 :** L'intégration PCIe est historiquement capricieuse sur ce SoC. Le contrôleur gère mal l'ASPM (Active State Power Management) et l'IOMMU lors des requêtes DMA du Coral. Pour éviter des *kernel panics* ou des timeouts du bus, il est souvent impératif de forcer `pcie_aspm=off` dans les paramètres de boot du noyau.

## 2. Le point de rupture : Python 3.14

Ubuntu 26.04 est livré avec Python 3.14 comme interpréteur par défaut. C'est ici que la grande majorité des forks de PyCoral s'effondrent.

L'API `pycoral` repose sur `pybind11` pour lier le code C++ au code Python. Depuis Python 3.12, des API C fondamentales (notamment celles gérant les threads internes) ont été supprimées. À moins que le fork `coralnpu` n'ait intégralement réécrit la surcouche `pybind11` pour s'adapter à Python > 3.12, la compilation plantera inévitablement sur des erreurs de type *implicit declaration of function*.

**Stratégie backend recommandée :**
Pour des opérations d'infrastructure pures, il est largement préférable d'esquiver la dette technique de `pycoral`. La méthode la plus robuste consiste à :

1. Compiler **uniquement** la librairie native `libedgetpu.so` en C++.
2. Utiliser la librairie standard `tflite-runtime` (ou `ai-edge-litert`), qui est maintenue pour Python 3.14.
3. Charger le TPU dynamiquement en tant que *External Delegate* dans votre code Python.

## 3. Prérequis et Packages (Ubuntu 26.04 aarch64)

Pour lancer la compilation des pilotes et de la librairie utilisateur depuis les sources, voici les dépendances strictes à provisionner :

**Pour le module noyau (Pilote PCIe) :**

```bash
sudo apt update
sudo apt install build-essential dkms linux-headers-$(uname -r) git

```

**Pour la compilation de `libedgetpu` :**

```bash
sudo apt install clang flatbuffers-compiler libusb-1.0-0-dev cmake

```

**Attention au système de build (Bazel) :**
La compilation de la couche Edge TPU exige Bazel. Il ne faut **pas** installer le paquet `bazel` via APT, car la version d'Ubuntu sera trop récente et cassera les fichiers `WORKSPACE`. Il faut déployer le binaire aarch64 de **Bazelisk**, qui téléchargera à la volée la version exacte (généralement 5.x ou 6.x) ciblée par les sources du fork.
