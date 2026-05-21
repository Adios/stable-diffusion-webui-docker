FROM pytorch/pytorch:2.12.0-cuda13.0-cudnn9-runtime

# Added PIP_BREAK_SYSTEM_PACKAGES=1 to bypass the externally-managed-environment blocker
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_PREFER_BINARY=1 \
    PIP_BREAK_SYSTEM_PACKAGES=1

# libgl1 and libglib2.0-0 are absolutely required for opencv-python on Ubuntu 24.04
RUN apt-get update && apt-get install -y git libgoogle-perftools-dev libgl1 libglib2.0-0 && apt-get clean

ARG PUID=0
ARG PGID=0
ARG USER_HOME=/root
# set build args as container environment variables for entrypoint reference
ENV PUID=$PUID
ENV PGID=$PGID
ENV USER_HOME=$USER_HOME

# if user home does not exist, create it
RUN mkdir -p "$USER_HOME"

# home already exists, chown it
RUN chown -R "${PUID}:${PGID}" "$USER_HOME"

# Only groupadd if we're non root
RUN if [ "$PGID" -ne "0" ]; then \
      echo "non root group detected"; \
      groupadd \
        --gid "$PGID" \
        stablediffusion ;\
    else \
      echo "root group detected" ; \
    fi

# Only useradd if we're non root
RUN if [ "$PUID" -ne "0" ]; then \
      echo "non root user detected"; \
      useradd \
        --gid="$PGID" \
        --no-user-group \
        -M \
        --home "$USER_HOME" \
        stablediffusion ; \
    else \
      echo "root group detected" ; \
    fi

USER $PUID:$PGID

ENV ROOT=/stable-diffusion

# Install ComfyUI and its basic requirements
RUN --mount=type=cache,target="$USER_HOME"/.cache/pip \
  git clone -b v0.20.1 --depth=1 https://github.com/comfyanonymous/ComfyUI.git ${ROOT} && \
  pip install --no-cache-dir -r ${ROOT}/requirements.txt

# ==============================================================================
# CORE CUSTOM NODE DEPENDENCY REGISTRY
# ==============================================================================
# The following Python modules are required by custom nodes mounted from volumes.
# Pre-baking them here avoids the Ubuntu 24.04 (PEP 668) background install block:
#
# - opencv-python   -> Required by: ComfyUI-Impact-Pack, ComfyUI-Impact-Subpack, 
#                                   ComfyUI-Inspire-Pack, ComfyUI_ControlNet_Aux
# - numba           -> Required by: WAS Node Suite (was-ns)
# - piexif          -> Required by: ComfyUI-Prompt-Reader-Node, ComfyUI-Image-Saver
# - matplotlib      -> Required by: ComfyUI-Florence2 (for parsing canvas masks/plots)
# - platformdirs    -> Required by: ComfyUI-Lora-Manager (handles cross-platform paths)
# - natsort         -> Required by: ComfyUI-Lora-Manager (natural alphanum sorting lists)
# - scikit-image    -> Required by: ComfyUI-Impact-Pack (advanced masking fallback)
# - einops          -> Required by: ComfyUI_ControlNet_Aux (tensor manipulations)
# ==============================================================================
RUN --mount=type=cache,target="$USER_HOME"/.cache/pip \
  pip install --no-cache-dir \
    opencv-python \
    scikit-image \
    numba \
    piexif \
    matplotlib \
    platformdirs \
    natsort \
    einops

# Install Triton (Mandatory dependency for SageAttention math execution)
RUN --mount=type=cache,target="$USER_HOME"/.cache/pip \
  pip install --no-cache-dir triton

# Install SageAttention directly via pip
RUN --mount=type=cache,target="$USER_HOME"/.cache/pip \
  pip install --no-cache-dir sageattention

# Install ComfyUI-Manager
RUN --mount=type=cache,target="$USER_HOME"/.cache/pip \
  git clone --depth=1  https://github.com/Comfy-Org/ComfyUI-Manager ${ROOT}/comfyui-manager && \
  pip install --no-cache-dir -r ${ROOT}/comfyui-manager/requirements.txt

RUN chown -R "$PUID:$PGID" "${ROOT}"

WORKDIR ${ROOT}
COPY --chown=$PUID:$PGID . /docker
RUN chmod u+x /docker/entrypoint.sh && cp /docker/extra_model_paths.yaml ${ROOT}

# NVIDIA runtime configuration, custom path logic, and TCMalloc memory linking
ENV NVIDIA_VISIBLE_DEVICES=all \
    PYTHONPATH="${PYTHONPATH}:${PWD}" \
    LD_PRELOAD="/usr/lib/x86_64-linux-gnu/libtcmalloc.so.4" \
    CLI_ARGS=""

EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD sh -c "python3 -c 'import os; print(f\"Running as UID={os.getuid()}, GID={os.getgid()}\")' && python -u main.py --listen --port 7860 ${CLI_ARGS}"
