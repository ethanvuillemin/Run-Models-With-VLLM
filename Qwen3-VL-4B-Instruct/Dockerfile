FROM vllm/vllm-openai:latest

# Définir les variables d'environnement
ENV MODEL_NAME="Qwen/Qwen3-VL-4B-Instruct"
ENV HOST="0.0.0.0"
ENV PORT="8000"

# Installer les dépendances supplémentaires si nécessaire
RUN pip install --no-cache-dir pillow requests

# Créer un répertoire pour les modèles
RUN mkdir -p /root/.cache/huggingface

# Exposer le port
EXPOSE 8000

# Commande de démarrage avec optimisations pour modèles de vision
# Nouvelle syntaxe vllm serve
CMD vllm serve ${MODEL_NAME} \
    --host ${HOST} \
    --port ${PORT} \
    --trust-remote-code \
    --max-model-len 4096 \
    --gpu-memory-utilization 0.9