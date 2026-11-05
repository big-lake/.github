terraform {
  required_version = ">= 1.5"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    time = {
      source = "hashicorp/time"
    }
  }

  backend "gcs" {
    bucket = "biglake-tf-state-test"
    prefix = "terraform/state"
  }
}

provider "google" {
  project = local.project_id
  region  = local.region

  # Required by the google_billing_budget resource (infra ADR-0008). Terraform
  # authenticates as a service account in CI, so the billingbudgets API call
  # has no consumer project to attribute the request to and returns a bare
  # 403 ("caller does not have permission") — distinct from a real IAM denial,
  # which carries an ErrorInfo permission name. Setting a billing_project +
  # user_project_override sends the X-Goog-User-Project header so the API
  # attributes/quota-checks against this project (where billingbudgets is
  # enabled and the deploy SA holds serviceusage.services.use). Only adds a
  # request header — produces no diffs on existing resources.
  billing_project       = local.project_id
  user_project_override = true
}

locals {
  env                       = "test"
  project_id                = "big-lake-test-490405"
  region                    = "australia-southeast2"
  zone                      = "australia-southeast2-b"
  github_org                = "big-lake"
  terraform_deploy_sa_email = "terraform-deploy@big-lake-test-490405.iam.gserviceaccount.com"
  # ADR infra/0001: apex zone is biglake.au (registrar delegates here
  # while prod doesn't exist yet). subdomain_prefix puts every test
  # surface under *.test.biglake.au so prod can claim the bare
  # *.biglake.au names later without collision.
  dns_zone         = "biglake.au"
  subdomain_prefix = "test"
  app_host         = "app.test.biglake.au"
  api_host         = "api.test.biglake.au"
  om_host          = "om.test.biglake.au"

  # Billing account that funds every BigLake project. The budget module
  # places an account-wide hard cap on it (infra ADR-0008).
  billing_account = "01B657-1B0524-72E3D0"
}

# Enable all required GCP APIs. Everything else depends on this module so
# that Terraform never races against an API not yet being enabled.
module "core" {
  source     = "../../modules/core"
  project_id = local.project_id
}

module "network" {
  source     = "../../modules/network"
  project_id = local.project_id
  env        = local.env
  region     = local.region
  depends_on = [module.core]
}

module "storage" {
  source        = "../../modules/storage"
  project_id    = local.project_id
  env           = local.env
  region        = local.region
  force_destroy = true

  knowledge_cors_origins = [
    "https://${local.subdomain_prefix}.${local.dns_zone}",
    "https://${local.app_host}",
    "http://localhost:5173",
  ]
}

module "iam" {
  source                          = "../../modules/iam"
  project_id                      = local.project_id
  env                             = local.env
  raw_bucket_name                 = module.storage.raw_bucket_name
  lakehouse_bucket_name           = module.storage.lakehouse_bucket_name
  api_bucket_name                 = module.storage.api_bucket_name
  knowledge_bucket_name           = module.storage.knowledge_bucket_name
  notebooks_public_bucket_name    = module.storage.notebooks_public_bucket_name
  intelligence_traces_bucket_name = module.storage.intelligence_traces_bucket_name
  semantics_bucket_name           = module.storage.semantics_bucket_name
  github_org                      = local.github_org
  terraform_deploy_sa_email       = local.terraform_deploy_sa_email
  # Grant org-domain developer accounts serviceAccountTokenCreator on the API SA
  # to call the OIDC-protected magic-link Cloud Function from local dev.
  # Note: org policy (constraints/iam.allowedPolicyMemberDomains) blocks @gmail.com.
  # Only add accounts from the permitted domain here.
  developer_token_creator_members = []
}

module "metastore" {
  source                     = "../../modules/metastore"
  project_id                 = local.project_id
  env                        = local.env
  region                     = local.region
  api_service_account_email  = module.iam.api_service_account_email
  data_service_account_email = module.iam.service_account_email
  # Lakehouse runtime catalog is created once via gcloud (see SETUP.md §11).
  # This module only manages the project-level IAM bindings for it.
  depends_on = [module.core, module.iam]
}

module "api" {
  source                  = "../../modules/api"
  project_id              = local.project_id
  env                     = local.env
  zone                    = local.zone
  service_account_email   = module.iam.api_service_account_email
  magic_link_function_url = module.email.function_url
  public_base_url         = "https://${local.api_host}"
  openmetadata_url        = "https://${local.om_host}/api/v1"
  # WebAuthn RP_ID stays at the eTLD+1 so the same passkey credential
  # works whether the user signs in via app.test.biglake.au (today) or
  # app.biglake.au (later, prod). Origins are per-env exact-match.
  webauthn_rp_id   = local.dns_zone
  webauthn_origins = "https://${local.app_host},https://${local.api_host}"
  # Exact UI origins the LB url_map routes to the ui backend (apex + app
  # subdomain) — api SEC-004/SEC-013. No wildcards.
  cors_origins                 = "https://${local.subdomain_prefix}.${local.dns_zone},https://${local.app_host}"
  security_policy              = module.cloud_armor.policy_self_link
  notebooks_public_bucket_name = module.storage.notebooks_public_bucket_name
  # Real bucket the knowledge pipeline promotes PDFs into (knowledge ADR-0012/0015) —
  # was never actually wired here (previously defaulted to the stale "big-lake-knowledge"
  # placeholder, 404ing every /knowledge/open signed URL).
  gcs_knowledge_bucket = module.storage.knowledge_bucket_name
  depends_on           = [module.email, module.auth_secrets]
}

module "ui" {
  source                = "../../modules/ui"
  project_id            = local.project_id
  env                   = local.env
  zone                  = local.zone
  service_account_email = module.iam.ui_service_account_email
  security_policy       = module.cloud_armor.policy_self_link
}

module "prefect" {
  source                = "../../modules/prefect"
  project_id            = local.project_id
  env                   = local.env
  zone                  = local.zone
  service_account_email = module.iam.service_account_email
}

module "openmetadata" {
  source                = "../../modules/openmetadata"
  project_id            = local.project_id
  env                   = local.env
  zone                  = local.zone
  service_account_email = module.iam.openmetadata_service_account_email
  # Use the OM-specific policy (600 req/min global limit) rather than the
  # shared api/ui policy (30 req/min). The OM SPA fires dozens of /api/v1/*
  # calls per page load; the tight shared limit 429s legitimate users.
  security_policy = module.cloud_armor.om_policy_self_link
}

module "knowledge_db" {
  source                = "../../modules/knowledge_db"
  project_id            = local.project_id
  env                   = local.env
  zone                  = local.zone
  service_account_email = module.iam.knowledge_db_service_account_email
  # Prefect (data SA), Flask BFF (api SA), and the RAG orchestrator
  # (intelligence SA) all connect to the DBs.
  client_service_accounts = [
    module.iam.data_service_account_email,
    module.iam.api_service_account_email,
    module.iam.intelligence_service_account_email,
  ]
}

module "intelligence" {
  source                = "../../modules/intelligence"
  project_id            = local.project_id
  env                   = local.env
  zone                  = local.zone
  service_account_email = module.iam.intelligence_service_account_email
  # Evaluation runs (evaluation repo run-eval.yml) call intelligence from CI through an IAP tunnel.
  extra_allowed_caller_sas = [module.iam.evaluation_service_account_email]
  # ADR-0028 arm (c) eval (grounding makes the compiler's raw-path fallback better); revert after.
  enable_semantic_grounding = true
  extra_env                 = { ENABLE_SEMANTIC_COMPILER = "true" }
}

# ---------------------------------------------------------------------------
# Passwordless auth stack (ADR api/0014, api/0015; infra ADR-0001).
# ---------------------------------------------------------------------------

module "auth_secrets" {
  source                           = "../../modules/auth_secrets"
  project_id                       = local.project_id
  env                              = local.env
  api_service_account_email        = module.iam.api_service_account_email
  api_deploy_service_account_email = module.iam.api_deploy_sa_email
}

module "email" {
  source                    = "../../modules/email"
  project_id                = local.project_id
  env                       = local.env
  region                    = local.region
  functions_source_bucket   = module.storage.api_bucket_name
  api_service_account_email = module.iam.api_service_account_email
  # mailer_user defaults to noreply@biglake.au (configured via Workspace DWD).
  depends_on = [module.core]
}

# Account-wide HARD spend cap with an automated kill switch (infra ADR-0008).
# Alert emails at 50%/90% (default IAM recipients); at 100% ACTUAL spend a
# Cloud Function disables billing on THIS project, taking it offline (prod
# runs its own instance of this module too — see environments/prod/main.tf —
# so together they cover every project on the account). Recovery is a manual
# Console re-enable — this is the intended, drastic behaviour of a genuine
# cap. The billing-account IAM grant it needs requires a one-off manual
# prerequisite (see SETUP.md).
#
# time_sleep: module.iam's pubsub.editor grant for the deploy SA (needed to
# create the topic below) can take a short time to propagate on GCP's side
# even after Terraform reports it created — without this, the very next
# apply step can hit a transient 403 on pubsub.topics.create. Only delays the
# FIRST apply that creates the grant; a no-op on every apply after.
resource "time_sleep" "wait_for_deploy_sa_iam" {
  depends_on      = [module.iam]
  create_duration = "60s"
}

module "budget" {
  source                  = "../../modules/budget"
  project_id              = local.project_id
  env                     = local.env
  region                  = local.region
  billing_account         = local.billing_account
  monthly_amount_aud      = 100
  functions_source_bucket = module.storage.api_bucket_name
  # Flip to false to test the notification wiring without disabling billing.
  kill_switch_armed            = true
  connect_pubsub_notifications = true
  depends_on                   = [module.core, time_sleep.wait_for_deploy_sa_iam]
}

# Cloud Armor security policy — rate limiting + optional AU geo-restriction.
# Attached to the api, ui, and openmetadata backend services via the
# security_policy variable. geo_restrict_au is false until the Newman CI
# job is migrated off GitHub's US-region runners (see SEC-002 in risks.md).
module "cloud_armor" {
  source          = "../../modules/cloud_armor"
  project_id      = local.project_id
  env             = local.env
  geo_restrict_au = false
  depends_on      = [module.core]
}

# Global HTTPS load balancer fronting api / ui / openmetadata
# (infra ADR-0001 §lb, Phase 1). Backends are the unmanaged instance
# groups each VM module now exposes. Managed cert covers all four
# hostnames (apex + app + api + om) and will only reach ISSUED after the
# registrar delegates biglake.au to Cloud DNS (manual step — see
# infra/SETUP.md).
module "lb" {
  source                       = "../../modules/lb"
  project_id                   = local.project_id
  env                          = local.env
  dns_zone                     = local.dns_zone
  subdomain_prefix             = local.subdomain_prefix
  ui_backend_service           = module.ui.backend_service
  api_backend_service          = module.api.backend_service
  openmetadata_backend_service = module.openmetadata.backend_service
  notebooks_public_bucket_name = module.storage.notebooks_public_bucket_name
  depends_on                   = [module.core]
}

# Cloud DNS — registers app/api/om/apex records under *.test.biglake.au
# (the apex zone itself is still biglake.au while prod doesn't exist).
module "dns" {
  source               = "../../modules/dns"
  project_id           = local.project_id
  env                  = local.env
  dns_zone             = local.dns_zone
  subdomain_prefix     = local.subdomain_prefix
  lb_ipv4_address      = module.lb.lb_ipv4_address
  dkim_key_placeholder = "" # populated manually after Workspace DKIM is generated
  depends_on           = [module.core]
}

# Phase 2 complete (merged jez/lb-phase-2 → main, 2026-05-25):
# Public IPs and 0.0.0.0/0 firewall rules removed from all three VMs.
# All services now reachable exclusively via the GCLB HTTPS endpoints.

output "wif_provider" {
  value = module.iam.wif_provider
}

output "metastore_rest_endpoint" {
  value       = module.metastore.rest_endpoint_uri
  description = "Lakehouse runtime catalog Iceberg REST endpoint"
}

output "metastore_catalog_name" {
  value       = module.metastore.catalog_name
  description = "Lakehouse runtime catalog name (= GCS analytics bucket name)"
}

output "metastore_warehouse" {
  value       = module.metastore.warehouse
  description = "Iceberg warehouse GCS path"
}
