```tf
locals {
  bucket_names_1 = ["cf2extr"]
  cf_name_1      = "core-iam-fun-gcp-0022331-default-01-${random_id.random_id_hex.hex}"
  cr_service_agent = [
    "roles/vpcaccess.user",
    "roles/iam.serviceAccountTokenCreator",
    "roles/run.serviceAgent"
  ]
  host_project_id = "wf-us-core-networking-hub-be54"
}


data "archive_file" "ref_source_zip" {
  type        = "zip"
  source_dir  = "${path.root}/file-cf2-extr/"
  output_path = "${path.root}/file-cf2-extr/function.zip"
}

# move archive into bucket
resource "google_storage_bucket_object" "archive_1" {
  name   = format("%s-%s.zip", local.cf_name_1, data.archive_file.ref_source_zip.output_md5)
  bucket = module.create_buckets_1.names[local.bucket_names_1[0]]
  source = data.archive_file.ref_source_zip.output_path
}


# Give access to needed resources
resource "google_storage_bucket_iam_member" "member_1" {
  bucket = module.create_buckets_1.names[local.bucket_names_1[0]]
  role   = "roles/storage.objectAdmin"
  # member = "serviceAccount:${local.executer_service_account.email}"
  member = "serviceAccount:sa-falcon-account@wf-us-core-iam-d53d.iam.gserviceaccount.com"
}

#Create cloud storage bucket for upload zip and output files
module "create_buckets_1" {
  source  = "tfe-nonprod.wellsfargo.net/TFE-GCP-SHARED/wf-cloud-storage-factory/google"
  version = "~> 5.0.0"
  # source = "git::https://github3.wellsfargo.com/tfe-gcp/terraform-google-wf-cloud-storage-factory/?ref=feature-wzmj-226"

############################
## Wells Fargo variables
############################
environment                = local.environment
app_id                     = local.app_id
au                         = local.au
data_classification_level  = local.data_classification_level
application_classification = local.application_classification
ci_environment             = local.ci_environment

############################
## Google variables
############################

  bucket_names = local.bucket_names_1
  encryption_kms_key_name = {
    "${local.bucket_names_1[0]}" = module.kms_bucket.key_names_to_key_id_map[local.bucket_key]
  }
  location      = local.location
  project_id    = local.project_id
  storage_class = "STANDARD"
  versioning    = {}
  force_destroy = { "${local.bucket_names_1[0]}" : true }
  depends_on = [
    module.kms_bucket
  ]
}

#Create HTTP Trigerred function
module "wf_cloudfunctions2_factory" {
  source = "tfe-nonprod.wellsfargo.net/TFE-GCP-shared/wf-cloud_functions_gen2-factory/google"
  version = "1.0.0-alpha.2"
  
  depends_on = [
    google_storage_bucket_iam_member.member
  ]
  
  # wf common Variables
  
  application_classification = local.application_classification
  ci_environment             = local.ci_environment
  instance_discriminator     = "cf2"
  
  # module specific variables
  region      = "us-central1"
  description = "Create reports and a service account extract file"
  project_id  = local.project_id
  
  GCR_parameters = {
    cmek_key          = module.kms_bucket.key_names_to_key_id_map[local.bucket_key]
    docker_repository = null
    autocreate        = true
  }
  
  service_environment_variables = {
    GCP_ORG_ID      = local.org_id,
    GCS_BUCKET_NAME = module.create_buckets_1.names[local.bucket_names_1[0]]
    CSV_OUTPUT_FILE = "extract.csv"
  }
  
  # Build Config parameters
  runtime       = "python39"
  entry_point   = "cf_entry_event"
  
  source_archive_bucket  = module.create_buckets_1.names[local.bucket_names_1[0]]
  source_archive_object  = google_storage_bucket_object.archive_1.name
  
  # Service Config Parameter
  max_instances                   = 5
  available_memory_mb             = "2048Mi"
  cf_runner_service_account_email = "sa-falcon-account@wf-us-core-iam-d53d.iam.gserviceaccount.com"
  timeout_in_seconds              = 3600
  #vpc_connector                  = "projects/${var.network_project_id}/locations/${var.project_id}/connectors/${var.vpc_connector}"
  #vpc_connector                   = "projects/wf-us-core-networking-hub-0ca8/locations/us-central1/connectors/core-con-gcp-usc1con"
  vpc_connector                   = "projects/wf-us-core-networking-hub-be54/locations/us-central1/connectors/core-con-gcp-usc1con"
  
  # Trigger the function:
  module "cf2_http_scheduler" {
    source  = "tfe-nonprod.wellsfargo.net/TFE-GCP/wf-scheduler-factory/google"
    version = "3.0.0" # Do not upgrade, this is the last HTTP cloud scheduler
############################
## Wells Fargo variables
############################

    environment                = var.environment
    app_id                     = "gcp"
    au                         = var.au
    data_classification_level  = var.data_classification_level
    application_classification = var.application_classification
    instance_discriminator     = "cf2${local.instance_discriminator_suffix}"
    
############################
## Google variables
############################
    project             = local.project_id
    region              = var.region
    schedule            = var.schedule
    time_zone           = local.time_zone
    description         = "Trigger iam-sa-extract cloud function"
    #message_to_publish = "Trigger Cloud Function"   # 3.0.0 version not had message_to_publish
    uri                 = module.wf_cloudfunctions2_factory.https_trigger_url
    http_method         = "POST"
    retry_count         = 1
    attempt_deadline    = "1800s"
    oidc_service_account_email = "sa-falcon-account@wf-us-core-iam-d53d.iam.gserviceaccount.com"
}

module "cr_service_agent" {
  source  = "localterraform.com/TFE-GCP-shared/wf-iam-factory/google//modules/grant-member-project-role"
  version = "~> 3.1.2"
  
  conditions            = []
  member                = "service-${local.project_number}@serverless-robot-prod.iam.gserviceaccount.com"
  member_account_type   = "serviceAccount"
  project_id            = local.project_id
  roles                 = local.cr_service_agent
}
```
