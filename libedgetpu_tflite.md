Comment charger dynamiquement libedgetpu.so via tflite_runtime en Python 3.14 sans avoir à compiler pycoral ? 



Esquiver `pycoral` est la meilleure décision d'architecture sur les versions récentes de Python. L'API `pycoral` n'est au fond qu'un *wrapper* de commodité. Le cœur du moteur d'inférence est TensorFlow Lite, qui possède un mécanisme natif de **Délégués Externes** (External Delegates).

Le binaire natif `libedgetpu.so` agit comme ce délégué matériel. Il suffit de le passer directement au *runtime* TFLite.

Voici comment implémenter cette solution proprement en Python 3.14.

## 1. Localiser la librairie partagée

Selon la méthode d'installation (paquet Debian ou compilation depuis les sources), la librairie compilée s'appelle généralement `libedgetpu.so.1.0` (souvent liée via un symlink `libedgetpu.so.1`).

Assurez-vous qu'elle se trouve dans un chemin standard de votre système (comme `/usr/lib/aarch64-linux-gnu/` ou `/usr/local/lib/`), ou définissez la variable d'environnement `LD_LIBRARY_PATH` pointant vers son répertoire avant de lancer le script Python.

## 2. Le code d'implémentation (Backend)

Vous aurez besoin du paquet Python `tflite_runtime` (ou de son nouveau nom poussé par Google, `ai-edge-litert`, si vous utilisez des *wheels* très récentes).

L'astuce consiste à utiliser la fonction `load_delegate` et à passer le résultat à l'argument `experimental_delegates` de l'interpréteur.

```python
import time
import numpy as np

# Gestion de la transition de nommage du paquet TensorFlow Lite
try:
    from tflite_runtime.interpreter import Interpreter, load_delegate
except ImportError:
    from ai_edge_litert.interpreter import Interpreter, load_delegate

# Nom ou chemin absolu vers la librairie partagée
EDGETPU_SHARED_LIB = 'libedgetpu.so.1'

def initialize_tpu_interpreter(model_path, device_index=None):
    """
    Charge le modèle TFLite et l'assigne au Coral Edge TPU.
    """
    print(f"Chargement du délégué : {EDGETPU_SHARED_LIB}")
    
    # Configuration des options du délégué (utile pour les Dual TPU)
    delegate_options = {}
    if device_index is not None:
        # Ex: ':0' pour le premier TPU, ':1' pour le second
        delegate_options['device'] = f':{device_index}'

    try:
        # 1. Charger le driver C++ natif
        tpu_delegate = load_delegate(EDGETPU_SHARED_LIB, options=delegate_options)
    except ValueError as e:
        print(f"Erreur fatale: Impossible de charger {EDGETPU_SHARED_LIB}.")
        print("Vérifiez que le module apex est chargé et que l'utilisateur est dans le groupe plugdev.")
        raise e

    # 2. Instancier l'interpréteur en lui injectant le délégué
    interpreter = Interpreter(
        model_path=model_path,
        experimental_delegates=[tpu_delegate]
    )

    # 3. Allouer la mémoire 
    # C'est à ce moment précis que le graphe TFLite est compilé et transféré au TPU.
    # Si le modèle n'est pas compilé pour Edge TPU (pas de suffixe _edgetpu.tflite),
    # il retombera silencieusement sur le CPU.
    interpreter.allocate_tensors()
    
    return interpreter

def run_inference(interpreter, input_data):
    """Exécute une passe d'inférence standard."""
    input_details = interpreter.get_input_details()
    output_details = interpreter.get_output_details()

    # Formater et injecter les données d'entrée
    interpreter.set_tensor(input_details[0]['index'], input_data)

    # Déclencher le TPU
    start_time = time.perf_counter()
    interpreter.invoke()
    inference_time = time.perf_counter() - start_time

    # Récupérer les résultats
    output_data = interpreter.get_tensor(output_details[0]['index'])
    
    return output_data, inference_time

# --- Exemple d'utilisation ---
if __name__ == "__main__":
    MODEL_PATH = "mon_modele_quantifie_edgetpu.tflite"
    
    # Initialisation
    interpreter = initialize_tpu_interpreter(MODEL_PATH)
    
    # Préparation d'un tenseur factice (à remplacer par vos données réelles)
    input_shape = interpreter.get_input_details()[0]['shape']
    dummy_input = np.zeros(input_shape, dtype=np.uint8)
    
    # Inférence
    result, duration = run_inference(interpreter, dummy_input)
    print(f"Inférence terminée en {duration * 1000:.2f} ms")

```

## 3. Ce que vous perdez sans `pycoral`

L'approche dynamique est la plus propre pour le système, mais elle vous prive de certaines fonctions utilitaires de `pycoral`. Vous devrez réécrire vous-même le code pour :

* Le redimensionnement automatique des images en entrée (souvent géré par OpenCV ou PIL de toute façon).
* L'extraction des boîtes englobantes (Bounding Boxes) post-inférence si vous faites de la détection d'objets (c'est de la manipulation matricielle pure avec Numpy).
* Le *pipelining* natif si vous vouliez scinder un modèle massif sur deux TPU (le Dual Edge TPU). Pour faire fonctionner les deux cœurs simultanément avec `load_delegate`, la méthode la plus fiable est d'instancier le script Python via du multiprocessing classique, en passant `device_index=0` à un processus et `device_index=1` à l'autre.
