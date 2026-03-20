# Лекция 8 — Terraform User Data: шпаргалка

EC2 + Security Group + nginx через User Data. Один `terraform apply` — и сайт работает.

---

## Шаг 0 — Подготовка

### Заполни terraform.tfvars

```powershell
# Windows (PowerShell):
Copy-Item terraform.tfvars.example terraform.tfvars
```

```bash
# Mac/Linux:
cp terraform.tfvars.example terraform.tfvars
```

Открой `terraform.tfvars` и замени значения:

```hcl
key_name    = "имя-твоего-ключа"   # AWS Console → EC2 → Key Pairs → столбец "Key pair name"
aws_profile = "имя-твоего-профиля" # профиль из Лекции 4
```

### Замени YOUR_ACCOUNT_ID в main.tf

Узнать Account ID:

```bash
aws sts get-caller-identity --query Account --output text --profile YOUR_AWS_PROFILE
```

Открой `main.tf` и замени строку:
```hcl
bucket = "terraform-state-YOUR_ACCOUNT_ID"
# →
bucket = "terraform-state-123456789012"  # твой реальный ID
```

---

## Шаг 1 — Запуск

```bash
terraform init
```
Ожидаемый результат: `Terraform has been successfully initialized`

```bash
terraform plan
```
Ожидаемый результат: `Plan: 2 to add` — Security Group и EC2

```bash
terraform apply -auto-approve
```
Ожидаемый результат: `Apply complete! Resources: 2 added.`

---

## ✅ Проверь себя — сайт работает

> Подожди 2–3 минуты после apply: EC2 стартует ~1 мин, nginx устанавливается ещё ~1–2 мин.

```bash
terraform output -raw server_url
```

Скопируй URL и открой в браузере — должна появиться страница **"Hello from Terraform!"**

---

## Шаг 2 — Посмотреть что создалось

```bash
# Все outputs сразу (URL, SSH команда, instance_id):
terraform output

# Список ресурсов которые Terraform отслеживает:
terraform state list
```

Ожидаемый вывод `state list`:
```
module.web_server.data.aws_ami.amazon_linux
module.web_server.aws_security_group.this
module.web_server.aws_instance.this
```

Проверить что state сохранился в S3:

```powershell
# Windows (PowerShell):
aws s3 ls --profile YOUR_AWS_PROFILE s3://terraform-state-YOUR_ACCOUNT_ID/lecture8/
```

```bash
# Mac/Linux:
aws s3 ls s3://terraform-state-YOUR_ACCOUNT_ID/lecture8/ --profile YOUR_AWS_PROFILE
```

---

## Шаг 3 — SSH подключение

```bash
# Получить готовую команду:
terraform output -raw ssh_command

# Скопируй и запусти её в терминале
```

Если nginx не отвечает — диагностика на сервере:

```bash
sudo systemctl status nginx
sudo cat /var/log/cloud-init-output.log
```

---

## Шаг 4 — Уничтожить ресурсы (обязательно!)

```bash
terraform destroy -auto-approve
```

Ожидаемый результат: `Destroy complete! Resources: 2 destroyed.`

### ✅ Проверь себя — ресурсы удалены

```bash
terraform state list
# Вывод должен быть пустым
```

Открой AWS Console → EC2 — инстанс должен исчезнуть (или быть в статусе "terminated").

---

## Troubleshooting

| Проблема | Решение |
|----------|---------|
| Сайт не открывается после apply | Подожди ещё 1–2 мин, nginx ещё устанавливается |
| `Connection refused` | Проверь Security Group — порт 80 должен быть открыт |
| Запрашивает `var.key_name` | Создай `terraform.tfvars` с `key_name = "имя-ключа"` |
| `Error acquiring the state lock` | `terraform force-unlock LOCK_ID` |
| `NoSuchBucket` | Проверь bucket name в `main.tf` — должен совпадать с реальным S3 bucket |

---

## Почитать

| Тема | Ссылка |
|------|--------|
| User Data на EC2 | https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html |
| aws_instance (user_data) | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#user_data |
| aws_security_group | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group |
| Heredoc `<<-EOF` | https://developer.hashicorp.com/terraform/language/expressions/strings#heredoc-strings |
