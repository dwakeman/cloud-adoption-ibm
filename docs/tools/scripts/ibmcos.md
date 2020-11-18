```sh
#!/bin/sh
#set -x
#-----------------------------------------------------------------------------------------------------
# Script:  ibmcos
# Version: 1.0.0
#-----------------------------------------------------------------------------------------------------
# IBM Cloud Object Storage - Utility functions
#-----------------------------------------------------------------------------------------------------
# Copyright (c) 2020, International Business Machines. All Rights Reserved.
#-----------------------------------------------------------------------------------------------------


cos_command="$1"
service_name="$2"
bucket_name="$3"
location_constraint="$4"
key_crn="$5"
region="us"
# Note:  Region should be set based on the location constraint, e.g. "us" for us-standard or 
#        "us-south" for us-south-standard

activity_tracker_crn="crn:v1:bluemix:public:logdnaat:us-south:a/9d5d52********************8a:55d90d9c-****-****-****-********785::"

#-----------------------------------------------------------------------------------------------------
# Display usage information for the commands in this script
#-----------------------------------------------------------------------------------------------------
display_usage()
{
	echo
    echo "NAME:"
    echo "ibmcos.sh - Manage features of IBM Cloud Object Storage"
    echo
    echo "USAGE:"
    echo "ibmcos.sh command [service instance name] [options]"
    echo
    echo "COMMANDS:"
    echo "-------------------------------------------------------------------------------------------"
    echo
    echo "view-buckets              View the buckets in the specified COS instance in XML format"
    echo "create-bucket             Create a new bucket with predefined configuration"
    echo "delete-bucket             Delete a bucket from the specified COS instance"
    echo "disable-bucket-firewall   Remove the COS Firewall configuration from a bucket"
	echo "view-bucket-config        Get the headers and configuration for the specified bucket"
	echo "view-objects              List the objects in the specified bucket"
	echo "location-constraints      Returns a list of valid location constraints for creating buckets"
	echo "view-key-protect          Returns a list of Key Protect instances"
	echo "view-keys-list            Returns a list of keys in the specified Key Protect instance"
	echo "help, h                   View help for this script"
    echo
    echo
    echo "Note: For your convenience this command executes the ibmcloud cli to look up certain"
    echo "      information needed to perform these tasks.  It requires you to be logged into"
    echo "      the ibmcloud cli before you run this command."
    echo
    echo
}

#-----------------------------------------------------------------------------------------------------
# Display the list of valid location constraints for creating buckets
#-----------------------------------------------------------------------------------------------------
display_location_constraints()
{
	echo
    echo "LOCATION CONSTRAINTS:"
    echo "-------------------------------------------------------------------------------------------"
    echo
    echo "us-south-standard       Regional Bucket in US South with storage class STANDARD"
    echo "us-east-standard        Regional Bucket in US East with storage class STANDARD"
    echo "us-standard             Cross regional Bucket in US Geo with storage class STANDARD"
    echo "us-south-flex           Regional Bucket in US South with storage class FLEX"
    echo "us-east-flex            Regional Bucket in US East with storage class FLEX"
    echo "us-flex                 Cross regional Bucket in US Geo with storage class FLEX"
    echo "us-south-cold           Regional Bucket in US South with storage class COLD"
    echo "us-east-cold            Regional Bucket in US East with storage class COLD"
    echo "us-cold                 Cross regional Bucket in US Geo with storage class COLD"
    echo "us-south-vault          Regional Bucket in US South with storage class VAULT"
    echo "us-east-vault           Regional Bucket in US East with storage class VAULT"
    echo "us-vault                Cross regional Bucket in US Geo with storage class VAULT"
	echo
	echo

}

#-----------------------------------------------------------------------------------------------------
# Display the headers and configuration of a bucket
#-----------------------------------------------------------------------------------------------------
view_bucket_configuration()
{
	echo
	echo "Bucket Headers:"
	echo
	
    curl -s -I "https://s3.${region}.cloud-object-storage.appdomain.cloud/${bucket_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'ibm-service-instance-id: '"$service_instance_guid"''

	echo
	echo "Bucket Configuration:"
	echo
	
    curl -s  "https://config.cloud-object-storage.cloud.ibm.com/v1/b/${bucket_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' | jq --color-output

	echo

}

#-----------------------------------------------------------------------------------------------------
# Validate the location constraint specified on a create-bucket request
#-----------------------------------------------------------------------------------------------------
validate_location_constraint()
{

	constraint="$1"

	#echo "The location constraint passed to the function is ${constraint}"

	locationCount=$(echo $constraints | jq '.locations | length')


	locationValid=false

	#echo "Value of locationValid is ${locationValid}"

	case $constraint in 
		us-standard)		locationValid=true;;
		us-south-standard)	locationValid=true;;
		us-south-flex)		locationValid=true;;
		us-south-cold)		locationValid=true;;
		us-south-vault)		locationValid=true;;
		us-east-standard)	locationValid=true;;
		us-east-flex)		locationValid=true;;
		us-east-cold)		locationValid=true;;
		us-east-vault)		locationValid=true;;
		us-standard)		locationValue=true;;
		us-flex)			locationValue=true;;
		us-cold)			locationValue=true;;
	esac

	#echo "Value of locationValid is ${locationValid}"

	if [[ $locationValid = false ]]; then
		echo
		echo "ERROR: ${constraint} is not a valid location constraint."
		echo
		display_location_constraints
		exit 1
	fi

}


#-----------------------------------------------------------------------------------------------------
# Check base input parameters
#-----------------------------------------------------------------------------------------------------
if [[ $# -lt 1  ]]; then	
	display_usage
    exit 1
fi


# Only need to do these commands if a service name is present
if [[ $# -ge 2 ]]; then
	# Get the GUID for the specified service instance
	service_instance_guid=$(ic resource service-instance $service_name --output JSON | jq -r '.[0].guid')

	# Get the bearer token to use for authentication.
	# Note:  This assumes that the user has previously logged into the IBM Cloud CLI
	ibm_auth_token=$(ic iam oauth-tokens | awk '{ print $4}')
fi


#-----------------------------------------------------------------------------------------------------
# View Help
#-----------------------------------------------------------------------------------------------------
if [ $cos_command == "help" ] || [ $cos_command == "h" ]; then

    echo
	display_usage
	exit 0
fi



#-----------------------------------------------------------------------------------------------------
# List all the buckets in the specified COS instance in XML format
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "view-buckets" ]]; then

    echo
    echo "Listing buckets for service ${service_name}..."
    echo
    
    curl -s "https://s3.${region}.cloud-object-storage.appdomain.cloud/?extended" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'ibm-service-instance-id: '"$service_instance_guid"'' | xmllint --format -


	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi


#-----------------------------------------------------------------------------------------------------
# View all the objects in the specified COS instance
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "view-objects" ]]; then
	if [[ $# -lt 3 ]]; then
		echo
		echo "for view-objects a bucket name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh view-objects [service instance name] [bucket-name]"
		echo
		echo
		exit 1
	fi

	echo
	echo "viewing objects for bucket ${bucket_name}..."
	echo
	
    curl -s "https://s3.${region}.cloud-object-storage.appdomain.cloud/${bucket_name}?List-type=2" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'ibm-service-instance-id: '"$service_instance_guid"''  | xmllint --format -

	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View all the objects in the specified COS instance
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "get-object" ]]; then
	if [[ $# -lt 3 ]]; then
		echo
		echo "for get-object a bucket name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh view-objects [service instance name] [bucket-name] [object-name]"
		echo
		echo
		exit 1
	fi

	if [[ $# -lt 4 ]]; then
		echo
		echo "for get-object an object name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh view-objects [service instance name] [bucket-name] [object-name]"
		echo
		echo
		exit 1
	fi
	
	object_name="$4"
	
	echo
	echo "getting object ${object_name}..."
	echo
	
    curl -s "https://s3.${region}.cloud-object-storage.appdomain.cloud/${bucket_name}/${object_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'ibm-service-instance-id: '"$service_instance_guid"'' > ${object_name}.pdf

	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi


#-----------------------------------------------------------------------------------------------------
# Create a bucket in the specified COS instance with Key Protect, Activity Tracker 
# and IP Firewall enabled
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "create-bucket" ]]; then
	if [[ $# -lt 3 ]]; then
		echo
		echo "for create-bucket a bucket name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh create-bucket [service instance name] [bucket-name] [location-constraint] [key-crn]"
		echo
		echo
		exit 1
	fi
	
	if [[ $# -lt 4 ]]; then
		echo
		echo "for create-bucket a location constraint is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh create-bucket [service instance name] [bucket-name] [location-constraint] [key-crn]"
		display_location_constraints
		echo
		exit 1
	fi
	
	if [[ $# -lt 5 ]]; then
		echo
		echo "for create-bucket a Key Protect Key is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh create-bucket [service instance name] [bucket-name] [location-constraint] [key-crn]"
		echo
		exit 1
	fi
	
	# Validate the specified bucket constraint
	validate_location_constraint $location_constraint
	
	if [[ $location_constraint == "us-standard" ]]; then
		region="us"
	fi
	
	echo "The updated region is ${region}"
	
	echo
	echo "Creating bucket ${bucket_name} with location constraint ${location_constraint}..."
	echo
	
    curl -s -X PUT "https://s3.${region}.cloud-object-storage.appdomain.cloud/${bucket_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'ibm-service-instance-id: '"$service_instance_guid"'' \
      -H 'ibm-sse-kp-encryption-algorithm: AES256' \
      -H 'ibm-sse-kp-customer-root-key-crn: '"$key_crn"'' \
      -d '<CreateBucketConfiguration><LocationConstraint>'"${location_constraint}"'</LocationConstraint></CreateBucketConfiguration>'

	# Next we need to update the config for Activity Tracker and COS Firewall
	# Note: A CIDR block of 0.0.0.0/0 allows all IP Addresses
	# Note: To remove the COS Firewall pass an empty array: {"allowed_ip": []}
		
    curl -s -X PATCH "https://config.cloud-object-storage.cloud.ibm.com/v1/b/${bucket_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -d '{ "activity_tracking": 
      		{
      			"read_data_events": false,
      			"write_data_events": true,
      			"activity_tracker_crn": "'"$activity_tracker_crn"'" 
      		},
			"firewall": 
			{
				"allowed_ip": ["0.0.0.0/0"] 
			}
      	  }' | jq --color-output


	echo
	echo "Bucket Created, retrieving bucket headers and configuration..."
	echo
	
	view_bucket_configuration
	
	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# Remove COS Firewall
# Note: This will probably only be used if the firewall config is invalid
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "disable-bucket-firewall" ]]; then
	if [[ $# -lt 3 ]]; then
		echo
		echo "for disable-bucket-firewall a bucket name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh disable-bucket-firewall [service instance name] [bucket-name]"
		echo
		echo
		exit 1
	fi
		
	echo
	echo "Disabling COS Firewall for bucket ${bucket_name}..."
	echo
	
    curl -s -X PATCH "https://config.cloud-object-storage.cloud.ibm.com/v1/b/${bucket_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -d '{ "firewall": {"allowed_ip": []} }' | jq --color-output

	echo
	echo "Bucket firewall disabled..."
	echo
	
	view_bucket_configuration
	
	echo "Request complete."
	echo
	echo
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# Delete a bucket
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "delete-bucket" ]]; then
	if [[ $# -lt 3 ]]; then
		echo
		echo "for delete-bucket a bucket name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh delete-bucket [service instance name] [bucket-name]"
		echo
		echo
		exit 1
	fi
		
	echo
	echo "Deleting bucket ${bucket_name}..."
	echo
	
    curl -s -X DELETE "https://s3.${region}.cloud-object-storage.appdomain.cloud/${bucket_name}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' 

	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi



if [[ $cos_command == "location-constraints" ]]; then

	display_location_constraints
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# Get Bucket Config
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "view-bucket-config" ]]; then
	if [[ $# -lt 3 ]]; then
		echo
		echo "for view-bucket-config a bucket name is required"
		echo
		echo "USAGE:"
		echo "ibmcos.sh view-bucket-config [service instance name] [bucket-name]"
		echo
		echo
		exit 1
	fi

	view_bucket_configuration
	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View a list of keys in the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "view-keys" ]]; then
    echo
    echo "Listing keys for service ${service_name}..."
    echo

	# Get the region for the Key Protect Instance
	location=$(ic resource service-instance $service_name --output json | jq -r '.[0].region_id')

    curl -s "https://${location}.kms.cloud.ibm.com/api/v2/keys" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key+json' | jq --color-output

	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi


#-----------------------------------------------------------------------------------------------------
# View a list of keys in the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "view-keys-list" ]]; then
    echo
    echo "Listing keys for service ${service_name}..."
    echo

	# Get the region for the Key Protect Instance
	location=$(ic resource service-instance $service_name --output json | jq -r '.[0].region_id')

    curl -s "https://${location}.kms.cloud.ibm.com/api/v2/keys" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key+json' | jq -r '[.resources | .[] 
         | {	name: .name, id: .id, crn: .crn} ] 
         | [ .[] | with_entries( .key |= ascii_downcase ) ] | (.[0] 
         | keys_unsorted | @tsv), (.[]|.| map(.) | @tsv) ' | column -t -s $'\t'

	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi


#-----------------------------------------------------------------------------------------------------
# View a list of keys in the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $cos_command == "view-key-protect" ]]; then
    echo
    echo "Listing instance of Key Protect..."
    echo

	

	ic resource service-instances --output JSON | jq -r '[ .[] 
	  | select(.sub_type != null) | select(.sub_type | contains("kms"))] 
	  | [ .[] | {name: .name, type: .type, sub_type: .sub_type }] 
	  | [ .[] | with_entries( .key |= ascii_downcase ) ] | (.[0] 
	  | keys_unsorted | @tsv), (.[]|.| map(.) | @tsv) ' | column -t -s $'\t'
	
	echo
	echo "Request complete."
	echo
	echo
	exit 0
fi


echo
echo "Error: Invalid command"
echo
display_usage
```




