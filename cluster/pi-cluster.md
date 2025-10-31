kubectl create secret generic ssh-key-secret \
  --from-file=id_ed25519=/home/chris/.ssh/id_ed25519 \
  --from-file=id_ed25519.pub=/home/chris/.ssh/id_ed25519.pub \
  -n cluster-namespace