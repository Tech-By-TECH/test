# create a user-mode netdev with optional SSH forward
{ 
  echo 'netdev_add user,id=net1,hostfwd=tcp::2222-:22';
  # add a virtio NIC on the hot-plug slot created earlier
  echo 'device_add virtio-net-pci,netdev=net1,id=nic1,bus=hp0';
  # verify
  echo 'info network';
  echo 'info pci';
} | nc 127.0.0.1 55555


packer-build:
  image: hashicorp/packer:latest
  services:
    - name: nginx:alpine
      alias: web
    - name: postgres:16-alpine
      alias: db
      command: ["-c", "listen_addresses='*'"]
  variables:
    PACKER_LOG: "1"
  script:
    # The services are on the same job network:
    - apk add --no-cache curl postgresql-client
    - curl -sSf http://web:80/        # works
    - psql -h db -U postgres -c 'select 1;' || true

    # Run Packer; inside your templates, use web/db hostnames above
    - packer init .
    - packer build .


.default-podman-remote:
  image: quay.io/podman/stable:latest
  variables:
    # Use SSH transport to the rootless user’s socket
    CONTAINER_HOST: "ssh://ci@runner-host/run/user/1001/podman/podman.sock"
    # If you use host key checking, also provide known_hosts or StrictHostKeyChecking=no
  before_script:
    - mkdir -p ~/.ssh
    - echo "$CONTAINER_SSHKEY" > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - printf 'Host runner-host\n  HostName runner-host\n  User ci\n' >> ~/.ssh/config
    - podman info

create-network:
  stage: prepare
  extends: .default-podman-remote
  script:
    - podman network create "ci-$CI_PIPELINE_ID" || true
    - podman network ls

job-a:
  stage: test
  needs: ["create-network"]
  extends: .default-podman-remote
  script:
    - podman run -d --name api --network "ci-$CI_PIPELINE_ID" docker.io/library/nginx:alpine
    - podman ps --format "table {{.Names}}\t{{.Networks}}"

job-b:
  stage: test
  needs: ["create-network"]
  extends: .default-podman-remote
  script:
    - podman run --rm --network "ci-$CI_PIPELINE_ID" quay.io/library/busybox ping -c1 api

cleanup:
  stage: cleanup
  when: always
  needs: ["job-a","job-b"]
  extends: .default-podman-remote
  script:
    - podman rm -f api || true
    - podman network rm "ci-$CI_PIPELINE_ID" || true
