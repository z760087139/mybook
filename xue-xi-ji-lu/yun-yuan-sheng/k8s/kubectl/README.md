# kubectl

自定义字段输出

kubectl get pod -o custom-columns=Name:metadata.name

custom-columns 内容为 "字段名:json路径"
