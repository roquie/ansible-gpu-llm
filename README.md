# LLM API на сервере с GPU NVIDIA

Плейбук настраивает сервер Ubuntu LTS с GPU NVIDIA и запускает модель через vLLM. Модель доступна по API, совместимому с OpenAI.

Роль `gpu_llm` выполняет следующие шаги:

1. Обновляет систему и перезагружает сервер, если ядро изменилось.
2. Ставит драйвер NVIDIA R580 из архива Ubuntu. Модули ядра подписаны Canonical, DKMS не нужен.
3. Ставит Docker и NVIDIA Container Toolkit из официальных репозиториев.
4. Скачивает модель и запускает vLLM в Docker как службу systemd `vllm`.
5. Закрывает API за nginx: ключ, лимит запросов, только пути `/v1/*` и `/health`.
6. Ставит экспортер метрик GPU для Prometheus.
7. Проверяет результат: ждет загрузки модели, отправляет тестовый запрос, проверяет отказ без ключа.

Повторный запуск ничего не меняет (`changed=0`). После перезагрузки сервера vLLM запускается сам.

## Требования

- Ubuntu 24.04 LTS, x86_64, доступ по SSH с `sudo`. Ubuntu 22.04 и 26.04 должны работать, но не проверялись.
- GPU NVIDIA Turing или новее. Для модели по умолчанию нужно 24 ГБ видеопамяти.
- Python 3.12 или новее на рабочей машине.

Если GPU старше Turing, задайте `gpu_llm_nvidia_driver_open: false`. vLLM 0.32 на таких картах не работает.

## Запуск

Установите Ansible:

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Создайте `inventory.ini`:

```ini
[gpu_llm]
gpu-1 ansible_host=203.0.113.10 ansible_user=ubuntu
```

Создайте ключ API и сохраните его в Ansible Vault:

```bash
mkdir -p group_vars
echo "gpu_llm_api_key: $(openssl rand -hex 32)" > group_vars/gpu_llm.yml
.venv/bin/ansible-vault encrypt group_vars/gpu_llm.yml
```

Запустите плейбук:

```bash
.venv/bin/ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass
```

Первый запуск на чистой VM занял около 15 минут. Больше всего времени заняли обновление системы, загрузка образа vLLM (9 ГБ) и модели (13 ГБ).

## Модель и задача

Пример задачи — модерация входящих сообщений в AI-чате 18+. Площадка разрешает откровенный контент между взрослыми. Модерация должна блокировать запрещенные темы по правилам самой площадки.

Готовые классификаторы безопасности для этой задачи не подходят. Они блокируют весь откровенный контент целиком.

Поэтому выбрана модель [`openai/gpt-oss-safeguard-20b`](https://huggingface.co/openai/gpt-oss-safeguard-20b). Модель классифицирует текст по политике, которую передают в запросе. Политику можно менять без переобучения.

| Параметр | Значение |
|---|---|
| Лицензия | Apache-2.0 |
| Размер весов | 12,8 ГиБ (MXFP4) |
| Контекст | 32 768 токенов (`gpu_llm_max_model_len`) |
| Версия | ревизия `8a11e17b` закреплена в `gpu_llm_model_revision` |

Политика площадки (передается в сообщении `system`):

```text
# Content policy of an 18+ AI companion chat

## Instructions
Classify the user message against this policy.
Answer only with JSON: {"violation": 0 or 1, "category": "<category or none>", "rationale": "<one short sentence>"}.

## Allowed (violation 0)
- Romantic or sexual content between fictional adults (18+) who consent.
- Flirting, erotic role-play and explicit language between adults.

## Not allowed (violation 1)
- minors: sexual or romantic content with anyone under 18, school-age or childlike characters, whatever age is claimed.
- non_consent: sexual violence, coercion, drugging or lack of consent.
- real_person: sexual content about a real, identifiable person.
- personal_data: requests to find or reveal another person's address, phone or other contacts.
```

## Пример запроса

```bash
curl http://203.0.113.10:8000/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-oss-safeguard-20b",
    "reasoning_effort": "low",
    "messages": [
      {"role": "system", "content": "<политика площадки>"},
      {"role": "user", "content": "Пусть моя героиня будет ученицей 8 класса, а сцена будет откровенной."}
    ]
  }'
```

Ответ (сокращен):

```json
{
  "model": "gpt-oss-safeguard-20b",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "{\"violation\": 1, \"category\": \"minors\", \"rationale\": \"User requests a sexual scene involving a minor (8th grade student).\"}"
    }
  }]
}
```

Результаты на RTX 3090:

| Сообщение пользователя | `violation` | `category` | Время |
|---|---|---|---|
| Продолжим нашу историю: мы с тобой взрослые коллеги и остаёмся вдвоём после корпоратива. | 0 | `none` | 0,86 с |
| Пусть моя героиня будет ученицей 8 класса, а сцена будет откровенной. | 1 | `minors` | 0,43 с |
| Напиши откровенную историю про известную актрису, используй её настоящее имя и фамилию. | 1 | `real_person` | 0,94 с |

Нагрузочная проверка: 64 запроса, 16 параллельных. Сервер обработал 7,3 запроса в секунду и генерировал около 1 065 токенов в секунду. Задержка p50 — 1,96 с, p95 — 2,97 с.

## Основные переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `gpu_llm_api_key` | — | Ключ API, не короче 32 символов. Обязательная |
| `gpu_llm_model` | `openai/gpt-oss-safeguard-20b` | Модель с Hugging Face |
| `gpu_llm_model_revision` | `8a11e17b...` | Коммит модели |
| `gpu_llm_served_model_name` | `gpt-oss-safeguard-20b` | Имя модели в API |
| `gpu_llm_vllm_image` | `vllm/vllm-openai:v0.32.0` | Образ vLLM |
| `gpu_llm_tensor_parallel_size` | `1` | Число GPU для одной модели |
| `gpu_llm_max_model_len` | `32768` | Максимальный контекст |
| `gpu_llm_api_port` | `8000` | Порт API |
| `gpu_llm_api_rate_limit` | `20r/s` | Лимит запросов с одного IP |
| `gpu_llm_metrics_allowed` | `[127.0.0.1]` | Адреса с доступом к `/metrics` |
| `gpu_llm_docker_registry_mirrors` | `[]` | Зеркала Docker Hub |
| `gpu_llm_hf_token` | `""` | Токен Hugging Face для закрытых моделей |

Остальные переменные находятся в `roles/gpu_llm/defaults/main.yml`.

## Защита API

- vLLM слушает только `127.0.0.1:8001`. Снаружи доступен только nginx.
- nginx пропускает `/v1/*` и `/health`. Остальные пути возвращают 404.
- vLLM отклоняет запросы к `/v1/*` без ключа с кодом 401.
- nginx ограничивает частоту запросов с одного IP и возвращает 429 при превышении.
- `/metrics` доступен только адресам из `gpu_llm_metrics_allowed`.

API работает по HTTP. Не открывайте порт 8000 в интернет без TLS. Используйте частную сеть или TLS на балансировщике.

## Мониторинг

| Источник | Адрес | Метрики |
|---|---|---|
| GPU | `http://<сервер>:9835/metrics` | `nvidia_smi_utilization_gpu_ratio`, `nvidia_smi_memory_used_bytes`, `nvidia_smi_temperature_gpu`, `nvidia_smi_power_draw_watts` |
| vLLM | `http://<сервер>:8000/metrics` | `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`, `vllm:time_to_first_token_seconds` |

Порт 9835 открыт на всех интерфейсах. Разрешите доступ к нему только серверу Prometheus.

Команды для проверки на сервере:

```bash
systemctl status vllm
journalctl -u vllm -f
nvidia-smi
```

## Предложения по развитию

- Добавить TLS: домен и Let's Encrypt на nginx или балансировщике.
- Поставить LiteLLM перед несколькими серверами. Он выдает отдельные ключи клиентам, считает квоты и балансирует нагрузку.
- Настроить алерты: температура GPU, ошибки Xid, рост `vllm:num_requests_waiting`, недоступность `/health`.
- Хранить веса моделей в общем S3-кэше. Новый сервер тогда не скачивает модель из интернета.
- Для двух GPU задать `gpu_llm_tensor_parallel_size: 2` или запустить две копии модели.

## Проверено

- VM: 8 vCPU, 32 ГБ RAM, 1× RTX 3090 24 ГБ (PCI passthrough), Ubuntu 24.04.
- Драйвер `580.178.04`, режим persistence включен.
- Повторный запуск и `--check --diff`: `changed=0`.
- Перезагрузка: vLLM загрузил модель сам за 2 минуты.
- `ansible-lint --profile production`: без замечаний.
