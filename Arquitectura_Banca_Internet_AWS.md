# Arquitectura de Solución Cloud en AWS para Banca por Internet

**Versión:** 2025-10-31
**Formato:** Técnico detallado
**Autor:** Arquitecto de Soluciones Cloud

## Contenido
1. Resumen ejecutivo
2. Alcance y supuestos
3. Microservicios (detalle: API, endpoints, datos, escalado, SLAs)
4. Componentes AWS seleccionados y justificación técnica
5. Decisiones arquitectónicas (comparativas y razones)
6. Modelo C4 (Contexto, Contenedores, Componentes) - PlantUML incluídos
7. Diagramas adicionales: Secuencia, Infraestructura
8. Auditoría, retención y cumplimiento
9. Observabilidad y operaciones
10. Alta disponibilidad y DR (RTO/RPO)
11. Seguridad detallada (IAM, KMS, CloudHSM, mTLS, WAF)
12. Snippets IaC (VPC, Aurora, EKS)
13. Estimación de costos mensuales (rango aproximado)
14. Plan de pruebas operacionales y KPIs

---

## 1) Resumen ejecutivo
Se diseña una plataforma de Banca Digital a implementarse sobre el proveedor AWS, priorizando seguridad, alta disponibilidad, auditoría y cumplimiento regulatorio.
Plataforma principal: EKS (Kubernetes) para microservicios contenedorizados. Alternativa operacional: ECS/Fargate.
Base transaccional: Amazon Aurora (Postgres compatible). Eventos y orquestación: EventBridge, SQS, Step Functions.
Logs y auditoría: S3 (append-only) + Kinesis Firehose + OpenSearch + Athena para consultas.

## 2) Alcance y supuestos
- Integración con CORE on-premise mediante Direct Connect (o VPN redundante).
- IdP corporativo disponible (OAuth2/OIDC). Considerar Cognito solo si se internaliza IdP.
- Requisitos regulatorios: cifrado-at-rest/in-transit, segregación de red, auditoría inmutable.
- Tráfico estimado para dimensionamiento (ejemplo de referencia): 2000 TPS picos de API (ajustable).

## 3) Microservicios (detalle)
A continuación cada microservicio clave, responsabilidad, endpoints, persistencia y SLA.

### A. API Gateway / Edge
- Responsabilidad: enrutamiento, validación JWT, throttling(mecanismo que limita el número de solicitudes (requests)) y autenticación.
- Tecnologías: Amazon API Gateway (HTTP APIs) para endpoints públicos; ALB interno.
- SLA: p95 < 200ms para routing; alta disponibilidad multi-AZ.
- Escalado: gestionado (API Gateway) + EKS Horizontal Pod Autoscaler (HPA).

### B. Auth & Session Adapter
- Responsabilidad: delegación a IdP, validación de tokens, refresh, device binding.
- Persistencia: DynamoDB o ElastiCache para metadatos de sesión.
- Notas: usar IRSA (IAM Roles for Service Accounts) para permisos mínimos.

### C. User Onboarding & Identity
- Responsabilidad: flujo KYC, verificación facial, firmado de consentimientos.
- Integración: proveedores KYC (Onfido/Veriff) o Amazon Rekognition (si no hay requisitos regulatorios fuertes).
- Datos: PII cifrado con KMS CMK. Guardar solo hashes cuando sea posible.

### D. Accounts Service
- Responsabilidad: saldo, movimientos, consultas históricas (resumen y detalle).
- Persistencia: Aurora RDS (transaccional). Lecturas pesadas a replicas y caches (Redis).
- Modelo: usar CQRS; writes -> RDS, reads -> materialized views/replicas or read model in DynamoDB.

### E. Transfers & Payments Orchestrator
- Responsabilidad: orquestación de transferencias (validación, reserva, debit core, reconciliation).
- Tecnologías: Step Functions para flujos stateful; EventBridge + SQS para eventos/colas.
- Persistencia: RDS + events persisted to S3 for audit.

### F. Clearing Adapter / Switch
- Responsabilidad: conectividad con CORE y sistemas externos (ACH, SWIFT).
- Seguridad: mTLS, network restrictions, idempotency via DynamoDB.

### G. Beneficiaries / Standing Orders
- Responsabilidad: CRUD beneficiarios, plantillas de pago programado.
- Persistencia: DynamoDB para lectura rápida y TTL, RDS si se requieren relaciones complejas.

### H. Limits & Risk Engine
- Responsabilidad: reglas en tiempo real para límites, fraude y AML callouts.
- Integración: ML services as a microservice or external provider.
- SLA: alta disponibilidad, baja latencia (< 100ms lookups).

### I. Notifications Service
- Responsabilidad: enviar email, SMS y push (Pinpoint/SES/SNS).
- Persistencia: log de envíos en S3 + estado en DynamoDB.

### J. Audit & Compliance Service
- Responsabilidad: append-only audit logs; signing with KMS; query via Athena/OpenSearch.
- Pipeline: services -> Kinesis Firehose -> S3 (partitioned) -> Glue/Athena + OpenSearch for fast search.

### K. Reporting & Analytics
- Responsabilidad: BI, queries históricas.
- Tecnologías: Redshift spectrum or Athena over S3; ETL via Glue.

### L. Batch Processor / File Processor
- Responsabilidad: procesar cargas masivas (pagos por archivo).
- Tecnologías: AWS Batch or EKS Jobs; use S3 for file staging.

## 4) Componentes AWS seleccionados y justificación técnica
- Route53: resolución DNS y health checks (failover). Integración con Route53 latency-based routing para latencia global.
- CloudFront: CDN para SPA y assets estáticos, reduce latencia y mejora seguridad.
- API Gateway + WAF: protege Borde, throttling, JWT authorizers, y reglas OWASP en WAF.
- EKS + ECR: despliegue de microservicios, control granulado y service mesh (Istio/Linkerd) si se requiere.
- Aurora RDS: base transaccional con replicas y PITR.
- DynamoDB: sesiones, idempotency store, beneficiaries cache, TTLs.
- ElastiCache: cache y locks distribuidos.
- S3 + Kinesis Firehose + Athena + OpenSearch: pipeline de audit y analytics.
- EventBridge / SQS / Step Functions: orquestación y desacoplamiento.
- SES / Pinpoint: comunicaciones (email, SMS, push).
- CloudTrail + Config + GuardDuty: auditoría y detección de intrusos.
- KMS / CloudHSM: cifrado de claves sensibles; HSM si regulatorio.
- Direct Connect / VPN: conectividad segura con CORE on-prem.
- Secrets Manager: rotación de secretos y acceso programático seguro.
- AWS Backup: backups centralizados para RDS, EFS, etc.
- CloudWatch + X-Ray: observability, tracing, alarming.

## 5) Decisiones arquitectónicas (comparativas)
- EKS vs ECS/Fargate: EKS elegido por control y portabilidad; ECS/Fargate alternativa por reducción operativa.
- Aurora vs DynamoDB: Aurora para transaccionalidad fuerte; DynamoDB para key-value de alta concurrencia.
- EventBridge + Step Functions vs Kafka/MSK: EventBridge + Step Functions por integración AWS y simplicidad; MSK si necesidad de throughput masivo o retención stream compleja.
- Rekognition vs Onfido/Veriff: proveedor especializado elegido para KYC por certificación y anti-spoofing.
- KMS vs CloudHSM: KMS por la mayoría de casos; CloudHSM para claves no exportables o requisitos regulatorios.

## 6) Modelo C4 y PlantUML
Los archivos PlantUML incluidos en /docs/Diagramas (C4_Context.puml, C4_Containers.puml, C4_Components_Transfers.puml, Secuencia_Transferencia.puml, Infraestructura_AWS.puml).

## 7) Diagramas adicionales: Secuencia, Infraestructura 

## 8) Auditoría, retención y cumplimiento
- Events -> Firehose -> S3 with partitioning by date/service.
- Each event signed with HMAC using a KMS CMK (store key in KMS).
- OpenSearch indices for last 90 days, S3 Standard for 1 year, Glacier for long term.
- GDPR / local data laws: PII minimization, encryption, right to be forgotten procedures and data retention policies.
- PCI-DSS: tokenization, isolated VPC/account if card data is present.

## 9) Observabilidad y operaciones
- Logs (FluentBit) -> CloudWatch Logs + Firehose -> S3 + OpenSearch.
- Metrics: Prometheus in-cluster -> push to CloudWatch (or use CloudWatch agent).
- Traces: OpenTelemetry -> X-Ray/Jaeger.
- Dashboards: CloudWatch + Kibana (OpenSearch).
- Alerts: CloudWatch Alarms + SNS -> On-call (PagerDuty).

## 10) Alta disponibilidad y DR
- Multi-AZ deployment for RDS & EKS nodes across at least 3 AZs.
- Cross-Region read-replica for Aurora; snapshots and recovery playbooks.
- Route53 health checks for DNS failover.
- IaC versioned for reproducible rebuild (Terraform/CloudFormation).

## 11) Seguridad
- Least privilege IAM + IRSA for EKS pods.
- Mutual TLS for CORE connectivity.
- WAF, Shield, GuardDuty, Macie (for sensitive data discovery).
- CloudTrail logs immutable to S3 bucket with MFA delete enabled.

## 12) Snippets IaC (ver carpeta IaC_Snippets)
- vpc.yaml (CloudFormation simplified)
- aurora.tf (Terraform conceptual)
- eks.yaml (eksctl conceptual)

## 13) Estimación de costos mensuales (rango aproximado)
La estimación es indicativa y depende del tráfico, almacenamiento y retención. Valores en USD / mes.

- EKS control plane + worker nodes (3 x m6i.large) : 3 x $140 = $420 (nodes) + EKS control plane ~$0 (ajustar) -> estimate $500 - $1500
- RDS Aurora (db.r6g.large primary + 1 read replica) : $800 - $3000 (depende de size/IO)
- DynamoDB (sessions/idempotency): $50 - $500
- ElastiCache (redis small cluster): $100 - $800
- API Gateway (high throughput): $200 - $2000
- S3 + Kinesis Firehose + Athena (audit pipeline): $50 - $500
- OpenSearch (hot cluster): $300 - $2000
- Direct Connect (port + data transfer) or VPN: $200 - $1500
- SES / Pinpoint (email/sms): $20 - $500 (depends on volume)
- CloudWatch logs/metrics/traces: $100 - $1000
- Misc (ECR, NAT GW, LB, WAF, Backup): $200 - $2000

**Rango total estimado: USD 2,000 — 12,000 / mes** (para una instalación de tamaño medio).

## 14) Plan de pruebas operacionales y KPIs
- Tests: carga (k6/JMeter), chaos (simulate AZ loss), DR runbooks, pen tests, SCA.
- KPIs: latency p95, availability, transfer success rate, MTTR, RTO/RPO.

## 15) Conclusiones y próximos pasos
- Validar requisitos regulatorios y SLAs con negocio.
- Implementar POC mínimo (EKS cluster + API Gateway + Transfers service minimal) y test end-to-end.
- Iterar con pruebas de carga y ajustar sizing y costs.

---
Fin del documento principal.
