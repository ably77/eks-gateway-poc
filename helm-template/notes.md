helm template "gloo-mesh-agent" gloo-mesh-agent/gloo-mesh-agent -n gloo-system --version 2.2.6 --values values.yaml > agent-out1.yaml