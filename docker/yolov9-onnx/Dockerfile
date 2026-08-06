# Exports YOLOv9 to ONNX for Frigate's OpenVINO detector and ships the result as a
# scratch image, so it can be mounted read-only through a Kubernetes image volume
# instead of being copied into Frigate's config PVC by hand.
#
# Follows Frigate's documented export recipe:
# https://docs.frigate.video/configuration/object_detectors/#openvino-detector
#
# All three variants are built at 320x320. Frigate is optimised for 320 models — it
# crops a motion region and zooms in before inference — so 640 is not worth the cost
# here. Shipping t, s and m together makes switching variants a config change rather
# than a rebuild.
FROM python:3.11 AS build

RUN apt-get update \
    && apt-get install --no-install-recommends -y cmake libgl1 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:0.12.2 /uv /bin/

WORKDIR /yolov9
ADD https://github.com/WongKinYiu/yolov9.git .

# CPU-only torch ahead of requirements.txt, which would otherwise pull the CUDA build
# and roughly 2GB of wheels this export never touches
RUN uv pip install --system --index-url https://download.pytorch.org/whl/cpu torch torchvision
# yolov9 predates numpy 2
RUN uv pip install --system "numpy<2"
RUN uv pip install --system -r requirements.txt
RUN uv pip install --system onnx==1.18.0 onnxruntime onnx-simplifier==0.4.* onnxscript

# torch 2.6 flipped torch.load to weights_only=True, which cannot load these checkpoints
RUN sed -i "s/ckpt = torch.load(attempt_download(w), map_location='cpu')/ckpt = torch.load(attempt_download(w), map_location='cpu', weights_only=False)/g" models/experimental.py

ARG IMG_SIZE=320
RUN mkdir /out \
    && for SIZE in t s m; do \
         curl -sSLo "yolov9-${SIZE}.pt" \
           "https://github.com/WongKinYiu/yolov9/releases/download/v0.1/yolov9-${SIZE}-converted.pt" \
         && python3 export.py --weights "./yolov9-${SIZE}.pt" --imgsz "${IMG_SIZE}" --simplify --include onnx \
         && mv "yolov9-${SIZE}.onnx" "/out/yolov9-${SIZE}-${IMG_SIZE}.onnx"; \
       done

FROM scratch
COPY --from=build /out/ /
