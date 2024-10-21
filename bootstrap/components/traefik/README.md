apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-users
  namespace: kube-system
stringData:
  # htpasswd -nb admin something STrong
  usersFile: |
    admin:$apr1$1QJ1QJ1Q$1QJ1QJ1Q
type: Opaque
