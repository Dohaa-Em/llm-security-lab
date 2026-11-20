# Hardening commands (target 3)

Run with `kubectl` (or `sudo k3s kubectl` on K3s). Namespace: `llm-apps`.

## 1. Remove the cluster-admin binding
```bash
kubectl delete clusterrolebinding llm-agent-vulnerable
```
Check: the stolen token now returns `401 Unauthorized`.

## 2. Disable ServiceAccount token automount
```bash
kubectl patch deployment llm-agent -n llm-apps --patch '{"spec":{"template":{"spec":{"automountServiceAccountToken":false}}}}'
kubectl patch deployment rag-pipeline -n llm-apps --patch '{"spec":{"template":{"spec":{"automountServiceAccountToken":false}}}}'
```

## 3. Run as non-root
```bash
kubectl patch deployment llm-agent -n llm-apps --patch '{"spec":{"template":{"spec":{"containers":[{"name":"agent-app","securityContext":{"privileged":false,"runAsUser":1000,"runAsNonRoot":true,"allowPrivilegeEscalation":false}}]}}}}'
```

## 4. Disable dangerous tools
```bash
kubectl patch deployment llm-agent -n llm-apps --patch '{"spec":{"template":{"spec":{"containers":[{"name":"agent-app","env":[{"name":"ALLOW_SHELL_EXEC","value":"false"},{"name":"ALLOW_FILE_ACCESS","value":"false"}]}]}}}}'
```

## 5. Replace NodePort by ClusterIP
```bash
kubectl patch service rag-pipeline -n llm-apps -p '{"spec":{"type":"ClusterIP","ports":[{"port":8002,"targetPort":8002,"protocol":"TCP"}]}}'
kubectl patch service llm-agent -n llm-apps -p '{"spec":{"type":"ClusterIP","ports":[{"port":8003,"targetPort":8003,"protocol":"TCP"}]}}'
kubectl patch service n8n -n llm-apps -p '{"spec":{"type":"ClusterIP","ports":[{"port":5678,"targetPort":5678,"protocol":"TCP"}]}}'
```

## 6. Network policies
See `k8s/defenses/`. External access goes through a WireGuard tunnel to an EC2 bastion (see `docs/wireguard-bastion.example.conf`).
