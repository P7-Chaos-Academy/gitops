# Prompting the Cluster LLaMA Model

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: llama-prompt
  namespace: prompts
spec:
  template:
    spec:
      schedulerName: llama-scheduler
      hostNetwork: true
      restartPolicy: Never
      containers:
        - name: llama
          image: curlimages/curl:8.9.1
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
          command:
            - sh
            - -c
            - >
              curl --request POST
              --url http://$HOST_IP:8080/completion
              --header "Content-Type: application/json"
              --data '{"prompt": "I love distributed systems for these reasons:", "n_predict": 128, "ignore_eos": true, "temperature": 0}'
```
