```py
import os
import json
import codecs
import csv
import argparse
import base64
from datetime import datetime
import re
from google.cloud import asset_v1
from google.cloud import storage
from google.cloud import resourcemanager_v3
from google.api_core.exceptions import GoogleAPIError
import proto
import googleapiclient.errors

def get_orgs_and_folders_metadata():
    folderClient = resourcemanager_v3.FoldersClient()
    orgClient = resourcemanager_v3.OrganizationsClient()
    try:
        request = resourcemanager_v3.SearchFoldersRequest()
        response = folderClient.search_folders(request=request)
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error in retrieving Folders : {err}')
        exit(0)
    try:
        request1 = resourcemanager_v3.SearchOrganizationsRequest()
        response1 = orgClient.search_organizations(request=request1)
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error in retrieving Organizations : {err}')
        exit(0)
    serializable_assets_1 = [proto.Message.to_dict(asset) for asset in response]
    serializable_assets_2 = [proto.Message.to_dict(asset) for asset in response1]
    serializable_assets_1.extend(serializable_assets_2)
    return serializable_assets_1

def get_all_proj_metadata():
    client = resourcemanager_v3.ProjectsClient()
    try:
        request = resourcemanager_v3.SearchProjectsRequest()
        response = client.search_projects(request=request)
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error: {err}')
        exit(0)
    serializable_assets = [proto.Message.to_dict(asset) for asset in response]
    return serializable_assets

def get_all_sas(org_id):
    scope = f"organizations/{org_id}"
    asset_types = ['iam.googleapis.com/ServiceAccount']
    query = "NOT name:(sandbox OR nonprod)"
    client = asset_v1.AssetServiceClient()
    try:
        response = client.search_all_resources(request={
            "scope": scope,
            "query": query,
            "asset_types": asset_types
        })
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error: {err}')
        exit(0)
    serializable_assets = [proto.Message.to_dict(asset) for asset in response]
    return serializable_assets

def get_all_sas_esar(org_id):
    scope = f"organizations/{org_id}"
    asset_types = ['iam.googleapis.com/ServiceAccount']
    client = asset_v1.AssetServiceClient()
    try:
        response = client.search_all_resources(request={
            "scope": scope,
            "asset_types": asset_types
        })
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error: {err}')
        exit(0)
    serializable_assets = [proto.Message.to_dict(asset) for asset in response]
    return serializable_assets

def get_iam_policies(svc_account, org_id):
    scope = f"organizations/{org_id}"
    query = f"policy:{svc_account}"
    client = asset_v1.AssetServiceClient()
    try:
        response = client.search_all_iam_policies(request={
            "scope": scope,
            "query": query
        })
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error: {err}')
        exit(0)
    sa_permissions = {}
    for policy in response:
        sa_permissions["resource"] = policy.resource
        sa_permissions["project"] = policy.project
        sa_permissions["asset_type"] = policy.asset_type
        sa_permissions["organization"] = policy.organization
    return sa_permissions

def get_all_iam_policies(org_id):
    scope = f"organizations/{org_id}"
    client = asset_v1.AssetServiceClient()
    try:
        response = client.search_all_iam_policies(request={"scope": scope})
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f'API Error: {err}')
        exit(0)
    serializable_assets = [proto.Message.to_dict(asset) for asset in response]
    return serializable_assets

def upload_content_gcp_bucket(gcp_bucket, dest_filename, file_contents):
    """Uploads a file to the bucket by using it's contents"""
    storage_client = storage.Client()
    bucket = storage_client.bucket(gcp_bucket)
    blob = bucket.blob(dest_filename)
    blob.upload_from_string(file_contents)

def upload_file_gcp_bucket(gcp_bucket, dest_filename, source_file):
    """Uploads a file to the bucket."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(gcp_bucket)
    blob = bucket.blob(dest_filename)
    blob.upload_from_filename(source_file)

def import_json_as_dictionary(filename):
    try:
        codecs.open(filename, encoding="utf-8", errors="strict").readline()
        utf_8 = True
        utf_16 = False
    except UnicodeDecodeError:
        utf_8 = False
    
    if not utf_8:
        try:
            codecs.open(filename, encoding="utf-16-le", 
                        errors="strict").readline()
            utf_16 = True
        except UnicodeDecodeError:
            utf_16 = False
    
    if utf_16:
        with open(filename, 'r', encoding='utf-16-le') as json_file_handler:
            data = json_file_handler.read()
            json_contents = json.loads(data.encode().decode('utf-8-sig'))
    elif utf_8:
        with open(filename, 'r') as json_file_handler:
            data = json_file_handler.read()
            json_contents = json.loads(data)
    else:
        print("Unable to determine file encoding, it's not utf-8 or utf-16-le")
        exit(0)
    return json_contents

def get_uid_from_email(sa_email, all_sas_dictionary):
    for svc_account in all_sas_dictionary:
        if 'additional_attributes' in svc_account:
            current_email = svc_account['additional_attributes']['email']
            if current_email == sa_email:
                uid = svc_account['additional_attributes']['uniqueId']
                break
        elif 'additionalAttributes' in svc_account:
            current_email = svc_account['additionalAttributes']['email']
            if current_email == sa_email:
                uid = svc_account['additionalAttributes']['uniqueId']
                break
        else:
            uid = "gcp_owned"
    return uid

def get_au_from_projectId(proj_metadata, project_id, mode):
    au = '0223092'
    try:
        for proj in proj_metadata:
            if (mode == "remote" and project_id in proj['project_id']) or (mode == "local" and project_id in proj['projectId']):
                if 'au' in proj['labels']:
                    au = proj['labels']['au']
                    break
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f"API Error : {err}")
        print(f"Setting au to a default value")
        au = "0223092"
    if au == "":
        au = "0223092"
    return au

def get_appId_from_projectId(proj_metadata, project_id, mode):
    appId = 'GCP'
    try:
        for proj in proj_metadata:
            if (mode == "remote" and project_id in proj['project_id']) or (mode == "local" and project_id in proj['projectId']):
                if 'appId' in proj['labels']:
                    appId = proj['labels']['appId']
                    break
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f"API Error : {err}")
        print(f"Setting appId to a default value")
        appId = 'GCP'
    if 'gcp' in appId:
        appId = 'GCP'
    return appId

def get_folderType_from_projectId(proj_metadata, project_id, mode):
    folderType = ''
    try:
        for proj in proj_metadata:
            if (mode == "remote" and project_id in proj['project_id']) or (mode == "local" and project_id in proj['projectId']):
                if 'environment' in proj['labels']:
                    folderType = proj['labels']['environment']
                    break
    except (GoogleAPIError, googleapiclient.errors.HttpError) as err:
        print(f"API Error : {err}")
        print(f"Setting folderType to a default value")
        folderType = ''
    if folderType == '':
        folderType = ''
    return folderType

def get_policy_for_identity(identity_info, proj_metadata, mode, iam_policy=None, binding=None, org_id=None):
    principal_policy = {}
    if identity_info['sa_type'] == "serviceAccount" or org_id is None:
        if 'assetType' in iam_policy:
            rsc_type = iam_policy['assetType'].split('/')[-1]
        elif 'asset_type' in iam_policy:
            rsc_type = iam_policy['asset_type'].split('/')[-1]
        rsc_name = iam_policy['resource'].split('/')[-1]
        rsc = f"{rsc_type}_{rsc_name}"
        role = binding['role'].replace('roles/', '')
        au = get_au_from_projectId(proj_metadata, identity_info['project_id'], mode)
        principal_policy[identity_info['email']] = {
            "RECORD_TYPE": "S",
            "UNIQUE_ID": identity_info['uid'],
            "SOR": "",
            "ID_LOCATION": "Production",
            "NAME": "",
            "STATUS": "A",
            "PRIV_IND": "3",
            "CERT_TYPE": "AU",
            "CERT_ENTITY": au,
            "LAST_NAME": "",
            "EMP_ID": "",
            "TID": "",
            "AU": au,
            "OWNING_APPL": get_appId_from_projectId(proj_metadata, identity_info['project_id'], mode),
            "LAST_LOGIN": "",
            "EMAIL": identity_info['email'],
            "Entitlement": [f"{role}_{rsc}"],
            "PROJECT_ID": identity_info['project_id'],
            "FOLDER_TYPE": get_folderType_from_projectId(proj_metadata, identity_info['project_id'], mode)
        }
    else:
        client = asset_v1.AssetServiceClient()
        parent = f"organizations/{org_id}"
        analysis_query = asset_v1.IamPolicyAnalysisQuery()
        analysis_query.scope = parent
        analysis_query.identity_selector.identity = f"user:{identity_info['email']}"
        analysis_query.options.expand_groups = True
        analysis_query.options.output_group_edges = True
        try:
            print(f"******analysis_query ******:{analysis_query}")
            response = client.analyze_iam_policy(
                request={"analysis_query": analysis_query},timeout=300)
        except Exception as err:
            print(f"analysis_query user:{identity_info['email']}")
            print(f"Error while making call to analyze_iam_policy : {err}")
        for policy in proto.Message.to_dict(response)["main_analysis"]["analysis_results"]:
            rsc_type = policy['attached_resource_full_name'].split('/')[-2]
            rsc_name = policy['attached_resource_full_name'].split('/')[-1]
            role = policy['iam_binding']['role'].replace('roles/', '')
            rsc = f"{rsc_type}_{rsc_name}"
            if policy['identity_list']['group_edges']:
                group_name = policy['identity_list']['group_edges'][0]['source_node'].replace(':', '_')
                entitlement = f"{role}_{rsc}"
            else:
                entitlement = f"{role}_{rsc}"
            if rsc_type != "Policy":
                if identity_info['email'] in principal_policy:
                    principal_policy[identity_info['email']]["Entitlement"].append(entitlement)
                else:
                    au = get_au_from_projectId(proj_metadata, identity_info['project_id'], mode)
                    principal_policy[identity_info['email']] = {
                        "RECORD_TYPE": "S",
                        "UNIQUE_ID": identity_info['uid'],
                        "SOR": "",
                        "ID_LOCATION": "Production",
                        "NAME": "",
                        "STATUS": "A",
                        "PRIV_IND": "3",
                        "CERT_TYPE": "AU",
                        "CERT_ENTITY": au,
                        "LAST_NAME": "",
                        "EMP_ID": "",
                        "TID": "",
                        "AU": au,
                        "OWNING_APPL": get_appId_from_projectId(proj_metadata, identity_info['project_id'], mode),
                        "LAST_LOGIN": "",
                        "EMAIL": identity_info['email'],
                        "Entitlement": [entitlement],
                        "PROJECT_ID": identity_info['project_id'],
                        "FOLDER_TYPE": get_folderType_from_projectId(proj_metadata, identity_info['project_id'], mode)
                    }
    return principal_policy[identity_info['email']]

def get_identity_info(member):
    identity_info = {}
    ignored_sa_types = set(
        ('projectOwner', 'projectEditor', 'projectViewer', 'group'))
    colon_counter = member.count(':')
    if colon_counter == 1:
        sa_type, sa_name = member.split(':')
        sa_other = "notUsed"
    elif colon_counter == 2:
        sa_other, sa_type, sa_name = member.split(':')
    else:
        sa_name = member
        sa_type = member
        sa_other = "notUsed"
    if sa_type not in ignored_sa_types and sa_other != "deleted":
        if sa_type != "allUsers":
            f_name = sa_name.split('@')[0]
            l_name = sa_name.split('@')[0]
        else:
            f_name = l_name = sa_name
        identity_info['sa_type'] = sa_type
        identity_info['email'] = sa_name
        identity_info['first_name'] = f_name
        identity_info['last_name'] = l_name
        if '@' in sa_name:
            identity_info['project_id'] = sa_name.split('@')[1].split('.')[0]
    else:
        identity_info["sa_type"] = "notUsed"
    return identity_info

def get_description_from_email(sa_email, all_sas_dictionary):
    for svc_account in all_sas_dictionary:
        if 'additional_attributes' in svc_account:
            current_email = svc_account['additional_attributes']['email']
            if current_email == sa_email:
                if 'description' in svc_account:
                    if svc_account['description']:
                        sa_description = svc_account['description']
                        break
        elif 'additionalAttributes' in svc_account:
            current_email = svc_account['additionalAttributes']['email']
            if current_email == sa_email:
                if 'description' in svc_account:
                    if svc_account['description']:
                        sa_description = svc_account['description']
                        break
    else:
        sa_description = "no_description_found"
    return sa_description

def parse_assets_output(all_iam_policies_dictionary, all_sas_dictionary, all_proj_metadata, run_mode, gcp_org_id=None):
    output_dict = {}
    for iam_policy in all_iam_policies_dictionary:
        if (run_mode == "remote" and iam_policy["asset_type"] != "orgpolicy.googleapis.com/Policy") or (run_mode == "local" and iam_policy["assetType"] != "orgpolicy.googleapis.com/Policy"):
            if iam_policy["policy"]["bindings"]:
                for binding in iam_policy["policy"]["bindings"]:
                    for member in binding["members"]:
                        # print(member)
                        identity = get_identity_info(member)
                        if identity["sa_type"] != "notUser":
                            if identity["sa_type"] == "serviceAccount":
                                identity["uid"] = get_uid_from_email(identity["email"], all_sas_dictionary)
                                if identity["uid"] != "gcp_owned":
                                    identity_policy = get_policy_for_identity(
                                        identity,
                                        all_proj_metadata,
                                        run_mode,
                                        iam_policy=iam_policy,
                                        binding=binding,
                                        org_id=None)
                                    if identity["email"] in output_dict:
                                        output_dict[identity["email"]]["Entitlement"].append(
                                            identity_policy["Entitlement"][0])
                                    else:
                                        output_dict[
                                            identity["email"]] = identity_policy
    return output_dict


def act_file_251(dictionary, filename):
    header = ["RECORD_TYPE", "UNIQUE_ID", "SOR", "ID_LOCATION", "NAME", "STATUS", "PRIV_IND", "CERT_TYPE", "CERT_ENTITY", "LAST_NAME", "EMP_ID", "TID", "AU", "OWNING_APPL", "LAST_LOGIN"]
    csv_file = filename
    try:
        with open(csv_file, "w") as csvfile:
            writer = csv.writer(csvfile, delimiter="|")
            writer.writerow(header)
            for _s, sa_value in dictionary.items():
                record_type = sa_value["RECORD_TYPE"]
                unique_id = sa_value["UNIQUE_ID"]
                sor = sa_value["SOR"]
                id_location = sa_value["ID_LOCATION"]
                name = sa_value["EMAIL"]
                status = sa_value["STATUS"]
                priv_ind = sa_value["PRIV_IND"]
                cert_type = sa_value["CERT_TYPE"]
                cert_entity = sa_value["CERT_ENTITY"]
                last_name = sa_value["LAST_NAME"]
                emp_id = sa_value["EMP_ID"]
                tid = sa_value["TID"]
                au = sa_value["AU"]
                owning_appl = sa_value["OWNING_APPL"]
                last_login = sa_value["LAST_LOGIN"]
                writer.writerow([record_type, unique_id, sor, id_location, name, status, priv_ind, cert_type, cert_entity, last_name, emp_id, tid, au, owning_appl, last_login])
    except IOError:
        print("I/O error, can't write out CSV file")
        
def act_file_252(dictionary, filename, all_folders_metadata):
   csv_columns = [ "Entitlement", "UNIQUE_ID", "OWNING_APPL" ]
   header = ["RESOURCE_TYPE", "UNIQUE_ID", "SOR", "RESOURCE_LOCATION", "NAME", "STATUS", "PRIV_IND", "CERT_TYPE", "CERT_ENTITY", "DESCRIPTION", "OWNING_APPL"]
   csv_file = filename
   try:
       with open(csv_file, "w") as csvfile:
           writer = csv.writer(csvfile, delimiter="|")
           writer.writerow(header)
           role = "Role"
           sor = ""
           status = "A"
           priv_ind = "3"
           cert_type = "APPL"
           cert_entity = "GCP"
           description = ""
           for _sa, sa_value in dictionary.items():
               Entitlement = sa_value["Entitlement"]
               Entitlement = [item for item in Entitlement if not 'sandbox' in item]
               Entitlement = [item for item in Entitlement if not 'nonprod' in item]
               for i in Entitlement:
                   owning_application = "GCP" #sa_value["OWNING_APPL"]
                   unique_id = "_".join(i.split(":", 2)[1:2])
                   resource_location = i.split(":", 2)[-1].replace("(","").replace(")","")
                   resource_location = resource_location.split('@')[0].replace('@',"")
                   resource_location = get_resource_location(resource_location, i, all_folders_metadata)
                   name = i.split('@')[0].replace('@',"").replace("(","").replace(")","")
                   writer.writerow([role, unique_id, sor, resource_location, name, status, priv_ind, cert_type, cert_entity, description, owning_application])
   except IOError:
       print("I/O error, can't write out CSV file")

def act_file_253(dictionary, filename, all_folders_metadata):
   csv_columns = [ "Entitlement", "UNIQUE_ID", "OWNING_APPL" ]
   header = ["UNIQUE_ID", "SOR", "ID_LOCATION", "PRIV_IND", "ATTR_NAME1", "ATTR_VALUE1", "ATTR_NAME2", "ATTR_VALUE2", "ATTR_CONTROL", "LOCATION", "ENTITLEMENT_STATUS", "OWNING_APPLICATION"]
   csv_file = filename
   try:
       with open(csv_file, "w") as csvfile:
           writer = csv.writer(csvfile, delimiter="|")
           writer.writerow(header)
           sor = ""
           id_location = "Production"
           status = "A"
           priv_ind = "3"
           attr_name1 = "Role"
           attr_name2 = ""
           attr_value2 = ""
           attr_control = ""
           entitlement_status = "Y"
           for _sa, sa_value in dictionary.items():
               Entitlement = sa_value["Entitlement"]
               Entitlement = [set(list(Entitlement))]
               Entitlement = [item for item in Entitlement if not 'sandbox' in item]
               Entitlement = [item for item in Entitlement if not 'nonprod' in item]
               for i in Entitlement:
                   owning_application = sa_value["OWNING_APPL"]
                   unique_id = sa_value["UNIQUE_ID"]
                   attr_value1 = "_".join(i.split(":", 2)[1:2]).replace('\'','').replace('\'','')
                   location = i.split(":", 2)[-1].replace("(","").replace(")","")
                   location = location.split('@')[0].replace('@',"")
                   location = get_resource_location(location, i, all_folders_metadata)
                   name = i.split('@')[0].replace('@',"").replace("(","").replace(")","")
                   writer.writerow([unique_id, sor, id_location, priv_ind, attr_name1, attr_value1, attr_name2, attr_value2, attr_control, location, entitlement_status, owning_application])
   except IOError:
       print("I/O error, can't write out CSV file")
       
def act_file_255(dictionary, filename):
    csv_columns = []
    header = ["DATE_STAMP", "OWNING_APPLICATION", "HASH_ALGORITHM", "FILE_FORMAT", "T251_HASH", "T251_ROWS", "T252_HASH", "T252_ROWS", "T253_HASH", "T253_ROWS", "T254_HASH", "T254_ROWS"]
    file_format = "ASCII-CRLF"
    csv_file = filename
    try:
        with open(csv_file, 'w') as csvfile:
            writer = csv.writer(csvfile, delimiter=',')
            writer.writerow(header)
            timestamp = datetime.today().strftime('%Y/%m/%d %H:%M:%S')
            omnin_app = "GCP"
            hash_algorithm = "SHA-256"

            count = 1
            t251_hash = count
            t251_rows = count
            for sa_value in dictionary.items():
                Entitlement = sa_value[1]['Entitlement']
                Entitlement = [item for item in Entitlement if not 'sandbox' in item]
                Entitlement = [item for item in Entitlement if not 'nonprod' in item]
                for i in Entitlement:
                    count += 1
                t251_rows = count

            count = 1
            t252_hash = count
            t252_rows = count
            for sa_value in dictionary.items():
                Entitlement = sa_value[1]['Entitlement']
                Entitlement = list(set(Entitlement))
                Entitlement = [item for item in Entitlement if not 'sandbox' in item]
                Entitlement = [item for item in Entitlement if not 'nonprod' in item]
                for i in Entitlement:
                    count += 1
                t252_rows = count

            count = 1
            t253_hash = count
            t253_rows = count
            for sa_value in dictionary.items():
                Entitlement = sa_value[1]['Entitlement']
                for i in Entitlement:
                    count += 1
                t253_rows = count

            t254_hash = count
            t254_rows = count

            writer.writerow([timestamp, omnin_app, hash_algorithm, file_format, t251_hash, t251_rows, t252_hash, t252_rows, t253_hash, t253_rows, t254_hash, t254_rows])
    except IOError:
        print("I/O error: can't write out CSV file")

def ESARextract(dictionary, filename, all_ss_asr_dictionary):
    header = ["Row Id", "Account Name", "AU", "Vendor Type", "Unique Object ID", "Tenant Type", "Folder Type",
              "Project Name", "Credential Type", "Primary Use", "Password Expiration Interval", "Is Interactive",
              "Privileged Account", "Business Justification", "Account Certification", "Application ID", "Security Plan",
              "Security Plan Exception"]
    csv_file = filename
    try:
        with open(csv_file, 'w') as csvfile:
            writer = csv.writer(csvfile, delimiter=',')
            writer.writerow(header)
            count = 0
            for sa_value in dictionary.items():
                print(sa_value[1]['AU'])
                au = sa_value[1]['AU']
                sa_desc = get_description_from_email(sa_value['EMAIL'], all_ss_asr_dictionary)
                match = re.search("AU-(.*?)", sa_desc)
                if match:
                    au = match.group(1).split("/")[0]
                else:
                    au = sa_value['AU']

                vendor_type = "Google Cloud Platform"
                unique_id = sa_value['UNIQUE_ID']
                tenant_type = sa_value['TENANT_LOCATION']
                project_name = sa_value['PROJECT_ID']
                if project_name == "seed-491f":
                    folder_type = "Bootstrap"
                elif project_name == "us-core-thirdpartytsaas-779a":
                    folder_type = "Local Account"
                    stf = sa_value['FOLDER_TYPE']
                    if stf.isdigit():
                        credential_type = "Systems Service Accounts"
                    else:
                        credential_type = "User Managed Service Accounts"
                else:
                    credential_type = "User Managed Service Accounts"

                # Extracting primary use from owning application
                primary_use = ""
                owning_app = sa_value['OWNING_APP']
                if "saas" in owning_app.lower():
                    primary_use = "Software as a Service (SaaS)"
                else:
                    primary_use = "Application"

                password_expiration_interval = "Non-Expiring"
                is_interactive = "FALSE"
                privileged_account = "FALSE"
                business_justification = "Used for provisioning folders/projects in GCP"
                account_certification = "ACR"
                application_id = sa_value['OWNING_APP']
                security_plan = ""
                security_plan_exception = ""

                if project_name == "ssappsvc" and project_name != "appsopt":
                    row_id = count
                    count += 1
                    writer.writerow([row_id, sa_value['NAME'], au, vendor_type, unique_id, tenant_type,
                                     folder_type, project_name, credential_type, primary_use,
                                     password_expiration_interval, is_interactive, privileged_account,
                                     business_justification, account_certification, application_id,
                                     security_plan, security_plan_exception])
    except IOError:
        print("I/O error, can't write out saExtract CSV file")

def write_roles_to_csv(filename, all_roles_metadata):
    header = ["Name", "Title", "Description"]
    csv_file = filename
    try:
        with open(csv_file, "w", newline="") as csvfile:
            writer = csv.writer(csvfile, delimiter=',')
            writer.writerow(header)
            count = 0
            for role in all_roles_metadata:
                count += 1
                name = role['name']
                if 'title' in role:
                    title = role['title']
                else:
                    print(f"Role title not available for {name}. Hence setting role name as title")
                    title = name
                if 'description' in role:
                    description = role['description']
                else:
                    print(f"Role description not available for {name}. Hence setting role name as Description")
                    description = name
                writer.writerow([name, title, description])

            print(f"Successfully wrote {count} roles data into file {csv_file}")

    except IOError:
        print("I/O error, can't write out IAM Roles CSV file")

def get_resource_location(resource_location, i, folder_metadata):
    location = resource_location
    res_location = "/".join(i.split("/")[1:]).rsplit("/", 2)[-2:]
    if "Project_" in res_location:
        location = get_foldername_from_id(folder_metadata, folder_metadata)
    elif "folders" in res_location:
        folders = res_location.replace("Project(", "").replace(")", "").replace("folders_", "folders/")
        location = get_foldername_from_id(folder_metadata, folders)
    elif "Folder_" in res_location:
        folders = res_location.replace("Project(", "").replace(")", "").replace("Folder_", "folders/")
        location = get_foldername_from_id(folder_metadata, folders)
    elif "organizations" in res_location:
        folders = res_location.replace("Project(", "").replace(")", "").replace("organizations_", "organizations/")
        location = get_foldername_from_id(folder_metadata, folders)
    elif "Organization_" in res_location:
        folders = res_location.replace("Project(", "").replace(")", "").replace("Organization_", "organizations/")
        location = get_foldername_from_id(folder_metadata, folders)
    elif "billingAccounts/01A624-0A0B41-1D4B20" in res_location:
        location = "Billing Account for GCP"
    elif "billingAccounts/03181F-C35908-DA82F" in res_location:
        location = "Billing Account for GCP"
    return location

def get_foldername_from_id(folder_metadata, folderName, location):
    folderDispName = location
    for folder in folder_metadata:
        if folderName in folder['name']:
            try:
                folderDispName = folder['display_name']
            except:
                folderDispName = folder['displayName']
            break
    return folderDispName


def write_dictionary_to_csv(dictionary, filename, all_folders_metadata, all_sas_esar_dictionary=None):
    if act_file_no == "file-251":
        act_file_251(dictionary, filename)
    elif act_file_no == "file-252":
        act_file_252(dictionary, filename, all_folders_metadata)
    elif act_file_no == "file-253":
        act_file_253(dictionary, filename, all_folders_metadata)
    elif act_file_no == "file-255":
        act_file_255(dictionary, filename)
    else:
        ESARextract(dictionary, filename, all_sas_esar_dictionary)


def cf_entry_event(event):
    print("Triggering Initiated")
    try:
        run_remote()
        return "Remote mode finished successfully"
    except:
        print("Remote mode failed")
        exit(0)


def cf_entry_http(request):
    print("This function was triggered by request {}".format(request))
    try:
        run_remote()
        return "Remote mode finished successfully"
    except:
        print("Remote mode failed")
        exit(0)

def run_local(iam_json_filename, sas_json_filename, esar_json_filename, proj_json_filename,
              roles_json_filename, folders_json_filename, csv_filename, gcs_bucket):
    print('Script running in local mode')
    
    all_iam_policies = import_json_as_dictionary(iam_json_filename)
    all_svc_accts = import_json_as_dictionary(sas_json_filename)
    all_svc_accts_esar = import_json_as_dictionary(esar_json_filename)
    all_proj_metadata = import_json_as_dictionary(proj_json_filename)
    all_roles_metadata = import_json_as_dictionary(roles_json_filename)
    all_folders_metadata = import_json_as_dictionary(folders_json_filename)

    merged_iam_sa_dictionary = parse_assets_output(all_iam_policies,
                                                   all_svc_accts, all_proj_metadata, "local")
    
    merged_iam_sa_dictionary_esar = parse_assets_output(all_iam_policies,
                                                        all_svc_accts_esar, all_proj_metadata, "local")

    file_no = ["251", "252", "253", "255", "eSARextract", "rolesMetaData"]
    for num in file_no:
        global act_file_no
        act_file_no = "file-" + num
        if num == "eSARextract":
            csv_filename = f"cf2-{num}" + ".csv"
            write_dictionary_to_csv(merged_iam_sa_dictionary_esar, csv_filename, all_folders_metadata, all_svc_accts_esar)
        elif num == "rolesMetaData":
            csv_filename = num + ".csv"
            write_roles_to_csv(csv_filename, all_roles_metadata)
        else:
            csv_filename = "cf2out-" + act_file_no + ".csv"
            write_dictionary_to_csv(merged_iam_sa_dictionary, csv_filename, all_folders_metadata)
            print(f"Wrote results to {csv_filename}")

    if gcs_bucket:
        upload_file_gcp_bucket(gcs_bucket, csv_filename, csv_filename)
        print(f"Uploaded file {csv_filename} to {gcs_bucket}")

def run_remote():
    print("Script running in remote mode")
    gcp_org_id = "796508071153"
    gcs_bucket = "core-iam-gcs-gcp-0222331-01-cf2ext"

    all_iam_policies = get_all_iam_policies(gcp_org_id)
    print("Print All IAM Policies:")
    print(all_iam_policies)

    all_svc_accts = get_all_sas(gcp_org_id)
    print("Print All Service Accounts:")
    print(all_svc_accts)

    all_svc_accts_esar = get_all_sas_esar(gcp_org_id)
    print("Print All Service Accounts - ESAR:")
    print(all_svc_accts_esar)

    all_proj_metadata = get_all_proj_metadata()
    print("Print All Project Metadata:")
    print(all_proj_metadata)

    all_folders_metadata = get_orgs_and_folders_metadata()
    print("Print All Folders Metadata:")
    print(all_folders_metadata)

    merged_iam_sa_dictionary = parse_assets_output(all_iam_policies, 
                                                   all_svc_accts, 
                                                   all_proj_metadata, 
                                                   gcp_org_id)

    merged_iam_sa_dictionary_esar = parse_assets_output(all_iam_policies, 
                                                        all_svc_accts, 
                                                        all_proj_metadata, 
                                                        all_svc_accts_esar, 
                                                        gcp_org_id)

    file_no = ["251", "252", "253", "255", "esarExtract"]
    for num in file_no:
        global act_file_no
        act_file_no = "file_" + num
        if num == "esarExtract":
            csv_filename = f"{num}.csv"
            csv_file_full_path = f"/tmp/{csv_filename}"
            write_dictionary_to_csv(merged_iam_sa_dictionary_esar, csv_file_full_path, all_folders_metadata, all_svc_accts_esar)
        else:
            csv_filename = f"C2out_{num}.csv"
            csv_file_full_path = f"/tmp/{csv_filename}"
            write_dictionary_to_csv(merged_iam_sa_dictionary, csv_file_full_path, all_folders_metadata)

        print(f"Wrote results to {csv_file_full_path}")
        upload_file_gcp_bucket(gcs_bucket, csv_file_full_path)
        print(f"Uploaded file {csv_file_full_path} to GCS")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    group = parser.add_mutually_exclusive_group()
    group.add_argument(
        "--remote",
        action="store_const",
        dest="mode",
        const="remote",
        help="run in remote mode reading from apis (default)"
    )
    group.add_argument(
        "--local",
        action="store_const",
        dest="mode",
        const="local",
        help="run in local mode reading from passed in json files"
    )
    parser.set_defaults(mode="remote")

    parser.add_argument("--gcs_bucket", help="upload results to gcs bucket")
    parser.add_argument("--iam_file", help="file containing all the iam policies (only in local mode)")
    parser.add_argument("--sas_file", help="file containing all the service accounts (only in local mode)")
    parser.add_argument("--sas_esar_file", help="file containing all the service accounts include sandbox and nonprod (only in local mode)")
    parser.add_argument("--proj_file", help="file containing all the projects metadata (only in local mode)")
    parser.add_argument("--role_file", help="file containing all the roles metadata (only in local mode)")
    parser.add_argument("--folders_file", help="file containing all the folders metadata (only in local mode)")
    parser.add_argument("--output_file", help="output file to write results to")

    args = parser.parse_args()

    if args.mode == "remote" and (args.iam_file or args.sas_file or args.sas_esar_file or args.proj_file or args.role_file or args.folders_file):
        print("as specified but local files are present")
        print("either switch to local mode or remove the file arguments")
        exit()

    if args.mode == "remote":
        run_remote()
    elif args.mode == "local":
        IAM_JSON_FILENAME = args.iam_file
        SAS_JSON_FILENAME = args.sas_file
        ESAR_SAS_JSON_FILENAME = args.sas_esar_file
        PROJ_JSON_FILENAME = args.proj_file
        ROLE_JSON_FILENAME = args.role_file
        FOLDERS_JSON_FILENAME = args.folders_file

        if args.output_file:
            CSV_FILENAME = args.output_file
        else:
            CSV_FILENAME = IAM_JSON_FILENAME.replace("json", "csv")

        GCS_BUCKET = args.gcs_bucket
        if not GCS_BUCKET:
            GCS_BUCKET = ""

        run_local(IAM_JSON_FILENAME, SAS_JSON_FILENAME, ESAR_SAS_JSON_FILENAME, PROJ_JSON_FILENAME, ROLE_JSON_FILENAME, FOLDERS_JSON_FILENAME, CSV_FILENAME, GCS_BUCKET)
```
