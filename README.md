# Kubernetes Webapp Deployment

Решение тестового задания.

## Особенности
- **Мультизональность**: `podAntiAffinity` по зонам - отказоустойчивость при сбое ноды
- **Инициализация**:  `readinessProbe` c `initialDelaySeconds: 12` -> защита от раннего трафика (5–10 сек инициализации).
- **Ресурсы**:
  - `requests.cpu: 100m`, `limits.cpu: 500m` — экономия + запас на старт.
  - `memory: 128Mi` — фиксировано.
- **Масштабирование**: HPA от 1 до 4 подов → минимум ресурсов ночью, максимум — днём.
- **Отказоустойчивость**: 4 пода справляются с пиком, распределены по зонам.

## Как проверить

```bash
minikube start --driver=docker --nodes=3
minikube addons enable metrics-server
kubectl apply -f .
kubectl get hpa -w
```

## Тех решения:
1. Отказоустойчивость: podAntiAffinity по зонам
     topologyKey: topology.kubernetes.io/zone

     - Поды распределяются по разным зонам
     - При падении одной зоны приложение остается доступно
     - Использовал preferredDuringScheduling для гибкости

2. Защита от раннего трафика: readinessProbe
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 12

    - Приложение инициализируется 5–10 сек → 12 секунд — безопасный запас.
    - Под не получает трафик, пока не станет Ready.

3. Автомасштабирование

    ```bash
    maxReplicas: 4
    targetCPUUtilization: 50%
    ```

   - Масштабируется от 1 (ночью) до 4 (днём) подов.
   - Цель — 50% загрузки CPU → баланс между производительностью и экономией.

4. Выделение ресурсов

   ```bash
     resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
   ```

   - requests.cpu: 100m — соответствует стабильной нагрузке.
   - limits.cpu: 500m — даёт "воздух" на старт.
   - Память стабильна — requests = limits.
   
