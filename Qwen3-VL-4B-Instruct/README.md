# Serveur vLLM avec Qwen3-VL-4B-Instruct

Guide complet pour déployer le modèle de vision Qwen3-VL-4B-Instruct avec vLLM et Docker.

## Prérequis

- Docker et Docker Compose installés
- NVIDIA GPU avec drivers CUDA
- NVIDIA Container Toolkit (`nvidia-docker2`)
- Au moins 16GB de VRAM GPU recommandés
- Au moins 8GB de RAM système

## Installation rapide

### 1. Vérifier le support GPU

```bash
# Vérifier que Docker peut accéder au GPU
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
```

### 2. Cloner et configurer

```bash
# Créer la structure de fichiers
mkdir vllm-qwen-server
cd vllm-qwen-server

# Copier les fichiers Dockerfile, docker-compose.yml et .env
# (voir les artifacts fournis)
```

### 3. Configuration optionnelle

Éditez le fichier `.env` si vous avez besoin d'un token Hugging Face pour des modèles privés.

### 4. Lancer le serveur

```bash
# Construire et démarrer
docker-compose up -d

# Voir les logs
docker-compose logs -f

# Le serveur sera prêt quand vous verrez:
# "Application startup complete"
```

## Utilisation

### Test simple

```bash
curl -X POST "http://localhost:8000/v1/chat/completions" \
  -H "Content-Type: application/json" \
  --data '{
    "model": "Qwen/Qwen3-VL-4B-Instruct",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "Describe this image in one sentence."
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://cdn.britannica.com/61/93061-050-99147DCE/Statue-of-Liberty-Island-New-York-Bay.jpg"
            }
          }
        ]
      }
    ]
  }'
```

### Test avec Python

```python
import requests

response = requests.post(
    "http://localhost:8000/v1/chat/completions",
    json={
        "model": "Qwen/Qwen3-VL-4B-Instruct",
        "messages": [{
            "role": "user",
            "content": [
                {"type": "text", "text": "Que vois-tu sur cette image?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://example.com/image.jpg"
                    }
                }
            ]
        }],
        "max_tokens": 512,
        "temperature": 0.7
    }
)

print(response.json()["choices"][0]["message"]["content"])
```

## Gestion du conteneur

```bash
# Arrêter le serveur
docker-compose down

# Redémarrer
docker-compose restart

# Voir les logs en temps réel
docker-compose logs -f vllm-server

# Vérifier le statut
docker-compose ps

# Nettoyer complètement (supprime aussi le cache du modèle)
docker-compose down -v
```

## Optimisations

### Pour GPU avec moins de VRAM

Modifiez le `Dockerfile` pour réduire la mémoire utilisée:

```dockerfile
CMD ["sh", "-c", "python -m vllm.entrypoints.openai.api_server \
    --model ${MODEL_NAME} \
    --host ${HOST} \
    --port ${PORT} \
    --trust-remote-code \
    --max-model-len 2048 \
    --gpu-memory-utilization 0.7"]
```

### Pour plusieurs GPUs

Modifiez `docker-compose.yml`:

```yaml
devices:
  - driver: nvidia
    count: all  # ou un nombre spécifique
    capabilities: [gpu]
```

## Dépannage

### Le conteneur ne démarre pas

```bash
# Vérifier les logs détaillés
docker-compose logs vllm-server

# Vérifier l'accès GPU
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
```

### Erreur de mémoire GPU

Réduisez `--gpu-memory-utilization` ou `--max-model-len` dans le Dockerfile.

### Le modèle télécharge lentement

Le premier démarrage peut prendre 10-30 minutes pour télécharger le modèle (environ 8GB). Les démarrages suivants seront rapides grâce au cache.

## API Compatible OpenAI

Le serveur expose une API compatible OpenAI à `http://localhost:8000/v1/`

Endpoints disponibles:
- `/v1/chat/completions` - Chat avec support multimodal
- `/v1/models` - Liste des modèles chargés
- `/health` - État du serveur

## Licence

Ce setup utilise vLLM (Apache 2.0) et Qwen3-VL-4B-Instruct. Vérifiez les licences des modèles avant usage en production.



---
---
---

Tu peux faire tourner uniquement les plus petits Qwen VL (2B/3B) avec ta config, et il faudra impérativement les quantizer (4 bits ou 8 bits) + limiter la résolution des images pour ne pas exploser tes 4 Go de VRAM.

​
Rappel de ta config

    OS : Windows.

​

RAM : 16 Go (OK pour petits modèles, limite pour gros contextes).

​

GPU : GTX 1650, 4 Go VRAM, CUDA OK (GPU d’entrée de gamme, bus mémoire étroit).

    ​

Cette config oblige à viser des modèles ≤ 3B fortement optimisés / offloadés vers la RAM CPU.

​
Qwen VL possibles avec 4 Go VRAM

Les docs Qwen2.5‑VL indiquent les besoins théoriques de VRAM par taille et quantization :

​

    Qwen2.5‑VL‑3B

        BF16 : ≈ 5.75 Go → trop pour 4 Go.

​

INT8 : ≈ 2.87 Go théoriques (compter 1.2× en vrai) → jouable si tu limites la taille d’image et le batch.

​

INT4 : ≈ 1.44 Go théoriques → beaucoup plus confortable, marge pour le contexte et l’overhead CUDA.

    ​

Qwen2.5‑VL‑7B

    Même en INT4, la doc recommande 8 Go VRAM mini, 16 Go pour être à l’aise → à oublier sur GTX 1650.

    ​

Qwen2‑VL‑2B / 3B (anciennes versions, similaires en taille)

    En pratique, on vise aussi 4 bits ou 8 bits, sinon OOM.

        ​

Conclusion :

    Chargeable et utilisable : Qwen2.5‑VL‑3B (ou Qwen2‑VL‑2B/3B) en INT4 ou INT8, avec offload partiel sur la RAM.

​

À éviter : tout ce qui est ≥ 7B (même INT4), ou les versions BF16/FP16 complètes.

    ​

Intérêt de la quantization (4b vs 8b)

Les benchmarks récents montrent :

​

    INT8

            99% des perfs d’origine sur les tâches vision‑langage.

    ​

Déjà un bon gain VRAM vs BF16, mais sur 4 Go ça reste serré si tu montes en contexte / résolution.

    ​

INT4

    ~ 98% de la perf sur les tâches vision, avec baisse légère surtout sur petits modèles.

​

Jusqu’à 4.3× plus rapide en throughput sur des VLMs, et énorme réduction mémoire (x2 vs INT8).

        ​

Sur une 1650 4 Go, INT4 est clairement le meilleur compromis :

    Assez de marge VRAM pour le modèle + le KV cache + un minimum de contexte.

​

Latence raisonnable malgré le GPU modeste.

    ​

Contraintes pratiques (images, vitesse, usage)

Même un « petit » Qwen2‑VL‑2B/3B peut OOM si :

​

    Tu passes des images très hautes résolutions (4K, scans A4 300 dpi, etc.).

    Tu laisses la lib gérer automatiquement la taille (max_pixels trop haut).

A faire absolument :

​

    Limiter la résolution :

        Longest side ≈ 1080–1280 px.

        Ou configurer max_pixels dans le processor.

    Utiliser torch_dtype=torch.float16 pour les activations, même avec poids INT4/INT8.

    Offload vers la RAM :

        Utiliser accelerate ou les options device_map="auto" avec une partie sur CPU (16 Go RAM suffisent pour un petit modèle 3B quantizé).

        ​

Attends‑toi à :

​

    Débit faible (quelques tokens/s) et forte latence image→texte.

    OK pour tests, prototypage, inference asynchrone, pas pour un chatbot temps réel avec gros contexte.

Reco concrètes

Pour ta machine actuelle :

    Modèle conseillé pour VL :

        Qwen2.5‑VL‑3B quantizé INT4 (ou Qwen2‑VL‑2B/3B en 4 bits type GGUF/GPTQ s’il existe une build adaptée).

    ​

Paramètres à viser :

    Quantization : 4 bits (INT4 / NF4) + compute en FP16.

    ​

    device_map="auto" avec offload sur CPU.

    Images max 1080–1280 px de côté, batch size 1.

À noter :

    Pour du texte‑only Qwen2.5, il existe des approches layer‑by‑layer qui permettent même du 72B sur 4 Go, mais c’est très lent et pas adapté à un usage interactif.

        ​

Si tu veux, tu peux préciser ton usage principal (OCR+QA, captioning, VQA multimodal, code etc.) et le runtime que tu préfères (Ollama, llama.cpp, vLLM, Transformers), et tu auras une reco plus ciblée (modèle exact + format + flags de lancement).



https://huggingface.co/helenai/Qwen3-VL-4B-Instruct-int4