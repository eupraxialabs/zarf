# Earthfile

ARG CONFIG="config.yaml"
ARG RHEL="false"

clean-build:
  LOCALLY
  RUN rm -fr build

# Used to load the RHEL7 RPMS
# earthly -s RHEL_USER=*** -s RHEL_PASS=*** +rhel-rpms 
rhel-rpms:
  FROM registry1.dso.mil/ironbank/redhat/ubi/ubi$RHEL
  WORKDIR /rpms

  RUN --secret RHEL_USER=+secrets/RHEL_USER --secret RHEL_PASS=+secrets/RHEL_PASS \
      subscription-manager register --auto-attach --username=$RHEL_USER --password=$RHEL_PASS
  
  RUN subscription-manager repos --enable=rhel-$RHEL-server-extras-rpms

  RUN yumdownloader --resolve --destdir=/rpms/ container-selinux

  # Download the K3S SELinux RPM 
  RUN curl -L "https://github.com/k3s-io/k3s-selinux/releases/download/v0.3.stable.0/k3s-selinux-0.3-0.el7.noarch.rpm" -o "/rpms/k3s-selinux.rpm"

  SAVE ARTIFACT /rpms

helm:
  FROM alpine/helm:3.5.3
  SAVE ARTIFACT /usr/bin/helm

yq:
  FROM  mikefarah/yq
  SAVE ARTIFACT /usr/bin/yq

charts:
  FROM registry1.dso.mil/ironbank/redhat/ubi/ubi8

  WORKDIR /src 

  COPY +yq/yq /usr/bin
  COPY +helm/helm /usr/bin
  COPY $CONFIG .

  RUN mkdir charts

  RUN yq e '.charts[] | .name + " " + .url' $CONFIG | \
      while read line ; do echo "repo add $line" | xargs -t helm; done

  RUN yq e '.charts[] | .name + "/" + .name + " -d ./charts --version " + .version' $CONFIG | \
      while read line ; do echo "pull $line" | xargs -t helm; done

  SAVE ARTIFACT /src/charts

k3s:
  FROM registry1.dso.mil/ironbank/redhat/ubi/ubi8
  WORKDIR /downloads

  COPY +yq/yq /usr/bin
  COPY $CONFIG /tmp/config.yaml

  RUN curl -fL "https://get.k3s.io" -o "init-k3s.sh"

  RUN K3S_VERSION=$(yq e '.k3s.version' /tmp/config.yaml) && \
      curl -fL "https://github.com/k3s-io/k3s/releases/download/$K3S_VERSION/{k3s,k3s-images.txt,sha256sum-amd64.txt}" -o "#1" && \
      sha256sum -c --ignore-missing "sha256sum-amd64.txt"

  SAVE ARTIFACT /downloads

images:
  FROM registry1.dso.mil/ironbank/google/golang/golang-1.16
  GIT CLONE --branch main https://github.com/google/go-containerregistry.git /go-containerregistry
  WORKDIR /go-containerregistry/cmd/crane

  COPY +yq/yq /usr/bin
  COPY $CONFIG .
  COPY +k3s/downloads/k3s-images.txt k3s-images.txt

  RUN k3s_images=$(cat "k3s-images.txt" | tr "\n" " ") && \
      app_images=$(yq e '.images | join(" ")' $CONFIG) && \
      images="$app_images $k3s_images" && \
      echo "Cloning: $images" | tr " " "\n " && \
      go run main.go pull $images /go/images.tar

  SAVE ARTIFACT /go/images.tar

compress: 
  FROM registry1.dso.mil/ironbank/redhat/ubi/ubi8
  WORKDIR /payload

  RUN yum install -y zstd

  COPY manifests manifests

  # Pull in artifacts from other build stages
  COPY +k3s/downloads bin
  COPY +charts/charts charts
  COPY +images/images.tar images/images.tar

  # Optional include RHEL rpm build step
  IF [ $RHEL != "false" ]
    COPY +rhel-rpms/rpms rpms
  END

  # Quick housekeeping
  RUN rm -f bin/*.txt && mkdir -p rpms

  RUN tar -cv . | zstd -T0 -16 -f --long=25 - -o /export.tar.zst

  SAVE ARTIFACT /export.tar.zst

build:
  FROM registry1.dso.mil/ironbank/google/golang/golang-1.16
  WORKDIR /payload

  # Pull in local assets
  COPY src .
  COPY +compress/export.tar.zst shift-package.tar.zst

  # Cache dep loading
  RUN go mod download 

  # Compute a shasum of the package tarball and inject at compile time
  RUN checksum=$(go run main.go checksum -f shift-package.tar.zst) && \
      echo "Computed tarball checksum: $checksum" && \
      go build -o shift-package -ldflags "-X shift/internal/utils.packageChecksum=$checksum" main.go

  # Validate the shasum before final packaging
  RUN ./shift-package validate

  RUN ls -lah shift-package*

  BUILD +clean-build
  SAVE ARTIFACT shift-package* AS LOCAL ./build/
