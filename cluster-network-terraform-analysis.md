# E2B Cluster Network Infrastructure - Terraform 深度分析

## 概述

`packages/cluster/network` 目录包含了 E2B 集群网络基础设施的完整 Terraform 配置。这是一个高度复杂的企业级网络架构，使用 Google Cloud Platform (GCP) 的全球负载均衡器、SSL 证书管理、DDoS 防护和细粒度的安全策略。

## 核心组件架构

### 1. Provider 配置

```hcl
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "4.19.0"
    }
  }
}
```

- 使用 **Cloudflare Provider** 管理 DNS 记录
- 通过 Google Secret Manager 安全存储 Cloudflare API Token

### 2. 后端服务定义

系统定义了 5 个核心后端服务：

#### Session Backend（会话服务）
```hcl
session = {
  protocol                        = "HTTP"
  port                            = var.client_proxy_port.port
  timeout_sec                     = 86400  # 24小时超时，支持长连接
  connection_draining_timeout_sec = 1
  http_health_check = {
    request_path       = var.client_proxy_health_port.path
    port               = var.client_proxy_health_port.port
    timeout_sec        = 3
    check_interval_sec = 3
  }
  groups = [{ group = var.api_instance_group }]
}
```
- **用途**：处理沙箱会话的 WebSocket 连接
- **特点**：超长超时时间（24小时），支持持久连接

#### API Backend（API 服务）
```hcl
api = {
  protocol                        = "HTTP"
  port                            = var.api_port.port
  timeout_sec                     = 65
  connection_draining_timeout_sec = 1
  # ...
}
```
- **用途**：处理 REST API 请求
- **特点**：标准 HTTP 超时配置

#### Docker Reverse Proxy Backend
```hcl
docker-reverse-proxy = {
  protocol    = "HTTP"
  port        = var.docker_reverse_proxy_port.port
  timeout_sec = 30
  # ...
}
```
- **用途**：Docker 镜像构建和管理的反向代理
- **特点**：中等超时时间

#### Nomad Backend
```hcl
nomad = {
  protocol    = "HTTP"
  port        = 80
  port_name   = "nomad"
  timeout_sec = 10
  # ...
}
```
- **用途**：Nomad 集群管理界面
- **特点**：短超时，快速响应

#### Consul Backend
```hcl
consul = {
  protocol    = "HTTP"
  port        = 80
  port_name   = "consul"
  timeout_sec = 10
  # ...
}
```
- **用途**：Consul 服务发现和配置管理
- **特点**：通过安全策略完全禁用外部访问

## SSL/TLS 证书管理

### 1. DNS 授权
```hcl
resource "google_certificate_manager_dns_authorization" "dns_auth" {
  name        = "${var.prefix}dns-auth"
  description = "The default dns auth"
  domain      = var.domain_name
  labels      = var.labels
}
```

### 2. 通配符证书
```hcl
resource "google_certificate_manager_certificate" "root_cert" {
  name        = "${var.prefix}root-cert"
  description = "The wildcard cert"
  managed {
    domains = [var.domain_name, "*.${var.domain_name}"]
    dns_authorizations = [
      google_certificate_manager_dns_authorization.dns_auth.id
    ]
  }
}
```
- 自动管理的 SSL 证书
- 支持主域名和所有子域名
- 支持多个额外域名配置

### 3. 证书映射
```hcl
resource "google_certificate_manager_certificate_map" "certificate_map" {
  name        = "${var.prefix}cert-map"
  description = "${var.domain_name} certificate map"
}
```

## URL 路由规则

### 主要路由配置
```hcl
resource "google_compute_url_map" "orch_map" {
  name            = "${var.prefix}orch-map"
  default_service = google_compute_backend_service.default["nomad"].self_link

  host_rule {
    hosts        = ["api.${var.domain_name}"]
    path_matcher = "api-paths"
  }

  host_rule {
    hosts        = ["docker.${var.domain_name}"]
    path_matcher = "docker-reverse-proxy-paths"
  }

  host_rule {
    hosts        = ["*.${var.domain_name}"]
    path_matcher = "session-paths"
  }
}
```

### 路由映射
- `api.domain.com` → API 后端
- `docker.domain.com` → Docker 反向代理
- `nomad.domain.com` → Nomad UI（带访问限制）
- `consul.domain.com` → Consul UI（完全禁用）
- `*.domain.com`（其他子域名） → 沙箱会话服务

## 负载均衡器配置

### 1. 全球 HTTPS 负载均衡器
```hcl
resource "google_compute_global_forwarding_rule" "https" {
  provider              = google-beta
  name                  = "${var.prefix}forwarding-rule-https"
  target                = google_compute_target_https_proxy.default.self_link
  load_balancing_scheme = "EXTERNAL_MANAGED"
  port_range            = "443"
}
```

### 2. 健康检查
```hcl
resource "google_compute_health_check" "default" {
  for_each = local.health_checked_backends
  
  check_interval_sec  = 5
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 2
  
  http_health_check {
    request_path = lookup(each.value["http_health_check"], "request_path", null)
    port         = lookup(each.value["http_health_check"], "port", null)
  }
}
```

### 3. 日志收集专用负载均衡器
```hcl
module "gce_lb_http_logs" {
  source  = "GoogleCloudPlatform/lb-http/google"
  version = "~> 12.1"
  name    = "${var.prefix}external-logs-endpoint"
  address = google_compute_global_address.orch_logs_ip.address
  # ...
}
```

## 安全策略

### 1. DDoS 防护
```hcl
resource "google_compute_security_policy" "default" {
  for_each = local.health_checked_backends
  
  dynamic "adaptive_protection_config" {
    for_each = each.key == "api" ? [true] : []
    
    content {
      layer_7_ddos_defense_config {
        enable = true
      }
    }
  }
}
```
- API 服务启用 Layer 7 DDoS 防护

### 2. 速率限制

#### API 速率限制 - 按 API Key
```hcl
resource "google_compute_security_policy_rule" "api-throttling-api-key" {
  action   = "throttle"
  priority = "300"
  
  rate_limit_options {
    conform_action = "allow"
    exceed_action  = "deny(429)"
    
    enforce_on_key_configs {
      enforce_on_key_name = "X-API-Key"
      enforce_on_key_type = "HTTP_HEADER"
    }
    
    rate_limit_threshold {
      count        = 1200
      interval_sec = 10
    }
  }
}
```
- **限制**：每个 API Key 10秒内最多 1200 个请求
- **应用于**：沙箱创建端点 (`/sandboxes` POST)

#### API 速率限制 - 按 IP
```hcl
rate_limit_threshold {
  count        = 20000
  interval_sec = 30
}
```
- **限制**：每个 IP 30秒内最多 20000 个请求

#### 沙箱连接速率限制
```hcl
# 按 Host Header
rate_limit_threshold {
  count        = 40
  interval_sec = 30
}

# 按 IP
rate_limit_threshold {
  count        = 40000
  interval_sec = 60
}
```

### 3. 访问控制
```hcl
resource "google_compute_security_policy_rule" "disable-consul" {
  action      = "deny(403)"
  priority    = "1"
  description = "Disable all requests to Consul"
  match {
    versioned_expr = "SRC_IPS_V1"
    config {
      src_ip_ranges = ["*"]
    }
  }
}
```
- Consul UI 完全禁止外部访问

## 防火墙规则

### 1. 健康检查防火墙
```hcl
resource "google_compute_firewall" "default-hc" {
  name    = "${var.prefix}load-balancer-hc"
  network = var.network_name
  source_ranges = [
    "130.211.0.0/22",  # Google 负载均衡器健康检查 IP 范围
    "35.191.0.0/16"
  ]
  target_tags = [var.cluster_tag_name]
}
```

### 2. SSH/RDP 访问控制
```hcl
resource "google_compute_firewall" "internal_remote_connection_firewall_ingress" {
  allow {
    protocol = "tcp"
    ports    = ["22", "3389"]
  }
  source_ranges = var.environment == "dev" ? ["0.0.0.0/0"] : ["35.235.240.0/20"]
}
```
- **开发环境**：允许所有 IP 访问
- **生产环境**：仅允许通过 Google IAP 访问

### 3. 出站流量
```hcl
resource "google_compute_firewall" "orch_firewall_egress" {
  allow {
    protocol = "all"
  }
  direction   = "EGRESS"
  target_tags = [var.cluster_tag_name]
}
```
- 允许所有出站流量

## Cloudflare DNS 集成

### 1. DNS 记录自动化
```hcl
resource "cloudflare_record" "a_star" {
  zone_id = data.cloudflare_zone.domain.id
  name    = "*"
  value   = google_compute_global_forwarding_rule.https.ip_address
  type    = "A"
  comment = var.gcp_project_id
}
```
- 自动创建通配符 A 记录
- 指向全球负载均衡器 IP
- 支持多域名配置

### 2. 证书验证记录
```hcl
resource "cloudflare_record" "dns_auth" {
  zone_id = data.cloudflare_zone.domain.id
  name    = google_certificate_manager_dns_authorization.dns_auth.dns_resource_record[0].name
  value   = google_certificate_manager_dns_authorization.dns_auth.dns_resource_record[0].data
  type    = google_certificate_manager_dns_authorization.dns_auth.dns_resource_record[0].type
  ttl     = 3600
}
```

## 变量配置

### 核心变量
- `prefix`: 资源名称前缀
- `environment`: 环境标识（dev/staging/prod）
- `domain_name`: 主域名
- `additional_domains`: 额外域名列表
- `cluster_tag_name`: 集群标签名称
- `network_name`: VPC 网络名称

### 端口配置
- `api_port`: API 服务端口配置
- `docker_reverse_proxy_port`: Docker 代理端口
- `client_proxy_port`: 客户端代理端口
- `logs_proxy_port`: 日志收集端口
- `nomad_port`: Nomad 服务端口

### 实例组配置
- `api_instance_group`: API 服务实例组
- `build_instance_group`: 构建服务实例组
- `client_instance_group`: 客户端服务实例组
- `server_instance_group`: 服务器实例组

## 输出值

```hcl
output "logs_proxy_ip" {
  value = google_compute_global_address.orch_logs_ip.address
}
```
- 输出日志收集服务的全球 IP 地址

## 架构特点总结

1. **全球化部署**
   - 使用 Google Cloud 全球负载均衡器
   - 支持多区域部署
   - Anycast IP 提供低延迟访问

2. **安全性**
   - 多层速率限制保护
   - DDoS 防护
   - 细粒度访问控制
   - 自动 SSL 证书管理

3. **可扩展性**
   - 基于实例组的自动扩缩容
   - 健康检查和自动故障转移
   - 连接排空确保优雅关闭

4. **监控和日志**
   - 独立的日志收集端点
   - 可配置的日志级别
   - 防火墙日志记录（生产环境）

5. **多租户支持**
   - 支持多个域名
   - 基于子域名的服务路由
   - 灵活的路径匹配规则

这个 Terraform 配置展示了一个生产级的云原生网络架构，充分利用了 GCP 的托管服务，同时保持了高度的安全性和可靠性。