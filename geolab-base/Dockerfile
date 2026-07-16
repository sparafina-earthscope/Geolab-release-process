# syntax=docker/dockerfile:1
# ──────────────────────────────────────────────────────────────
# Custom geolab-base -- built on pangeo/base-image
#
# Files picked up automatically by the parent's ONBUILD machinery:
#   environment.yml  → conda env "notebook" installed from this
#   conda-lock.yml   → (optional, preferred) no-solve install
#   start            → entrypoint script (becomes /srv/start)
#
# Build:
#   docker build --platform linux/amd64 \
#   --build-arg IMAGE_TITLE=my-geolab-image \
#   --build-arg IMAGE_AUTHORS=you@university.edu \
#   -t my-geolab-image .
# ──────────────────────────────────────────────────────────────
FROM pangeo/base-image:04bb14b

ARG IMAGE_TITLE="custom-geolab-image"   # default if user passes nothing
ARG IMAGE_AUTHORS="NoSpecifiedAuthors"  # default if user passes nothing

LABEL org.opencontainers.image.title="${IMAGE_TITLE}" \
      org.opencontainers.image.authors="${IMAGE_AUTHORS}"

# Record the released image version on disk. COPY writes as root regardless
# of the non-root USER the base image runs as, so this works without needing
# write access to /opt at runtime — CI overwrites this file with the real
# released version before `docker build` (see build-push.yml).
COPY VERSION /opt/VERSION

      # Set locations of PROJ/GDAL resource directories
ENV PROJ_DATA=/srv/conda/envs/notebook/share/proj \
    PROJ_LIB=/srv/conda/envs/notebook/share/proj \
    GDAL_DATA=/srv/conda/envs/notebook/share/gdal

# Default command for standalone use. JupyterHub spawning passes
# its own command (jupyterhub-singleuser), which start respects.
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--no-browser"]
