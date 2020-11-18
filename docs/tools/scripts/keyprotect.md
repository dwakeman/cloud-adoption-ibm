```sh
#!/bin/sh
#set -x
#-----------------------------------------------------------------------------------------------------
# Script:  keyprotect
# Version: v1.0.2
#-----------------------------------------------------------------------------------------------------
# IBM Key Protect - Utility functions
#-----------------------------------------------------------------------------------------------------
# Copyright (c) 2020, International Business Machines. All Rights Reserved.
#-----------------------------------------------------------------------------------------------------


display_usage()
{
	echo
    echo "NAME:"
    echo "keyprotect.sh - Manage features of Key Protect"
    echo
    echo "USAGE:"
    echo "keyprotect.sh <key-protect-instance-name> command [options]"
    echo
    echo "COMMANDS:"
    echo "-------------------------------------------------------------------------------------------"
    echo
    echo "view-policies           List the current policies for the Key Protect Instance"
    echo "enable-dual-auth        Enable the Dual Authorization policy for key deletes for all keys"
    echo "disable-dual-auth       Disable the Dual Authorization policy for key deletes for all keys"
    echo "disable-public-endpoint Disable the public endpoint for the Key Protect Instance"
    echo "enable-public-endpoint  Enable the public endpoint for the Key Protect Instance"
    echo "view-keys               List the keys in the Key Protect Instance in JSON format"
    echo "view-keys-list          List the keys in the Key Protect Instance in list format"
    echo "view-deleted-keys       List the deleted keys in the Key Protect Instance in JSON format"
    echo "view-deleted-keys-list  List the deleted keys in the Key Protect Instance in list format"
    echo "view-key                View the details of a key"
    echo "view-key-material       View the material for a standard key"
	echo "view-key-policies       View the current polices for the specified key"
	echo "import-key              Import a standard or root key"
	echo "restore-key             Restore an imported key that has been deleted"
    echo "set-key-deletion        Set the specified key for deletion (first auth)"
    echo "unset-key-deletion      Unset the specified key for deletion, which removes the first auth"
	echo "help, h                 View help for this script"
    echo
    echo
    echo "Note: For your convenience this command executes the ibmcloud cli to look up certain"
    echo "      information needed to perform these tasks.  It requires you to be logged into"
    echo "      the ibmcloud cli before you run this command."
    echo
    echo
}

view_policies()
{
    echo
    echo "Checking current policies for service ${service_name}..."
    echo

    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/instance/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'Content-Type: application/vnd.ibm.kms.policy+json' | jq --color-output
}

#-----------------------------------------------------------------------------------------------------
# View policies for a key
#
# Arguments: key-id
#    
#-----------------------------------------------------------------------------------------------------
view_key_policies()
{

	if [[ $# -lt 1 ]]; then
		echo
		echo "for view-key-policies a key id is required"
		echo
		echo "USAGE:"
		echo "keyprotect.sh <key-protect-instance-name] view-key-policies [key-id]"
		echo
		echo
		exit 1
	fi

	keyId=$1
	
    echo
    echo "Checking current policies for key ${keyId} in service ${service_name}..."
    echo

    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'Content-Type: application/vnd.ibm.kms.policy+json' | jq --color-output
}

#-----------------------------------------------------------------------------------------------------
# View keys in a Key Protect instance in JSON format
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
view_keys()
{
    echo
    echo "Listing keys for service ${service_name}..."
    echo

    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key+json' | jq --color-output

	echo
	echo "Request complete."
	exit 0

}

#-----------------------------------------------------------------------------------------------------
# View keys in a Key Protect instance in JSON format
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
view_deleted_keys()
{
    echo
    echo "Listing keys for service ${service_name}..."
    echo
    echo "Values for State field:"
    echo "-------------------------"
    echo "0  Pre-activation"
    echo "1  Active"
    echo "2  Suspended"
    echo "3  Deactivated"
    echo "5  Destroyed"
    echo
    
    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys?state=5" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key+json' | jq --color-output

	echo
	echo "Request complete."
	exit 0

}

#-----------------------------------------------------------------------------------------------------
# View a list of keys in a Key Protect instance
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
view_deleted_keys_list()
{
    echo
    echo "Listing deleted keys for service ${service_name}..."
    echo
    echo "Values for State column:"
    echo "-------------------------"
    echo "0  Pre-activation"
    echo "1  Active"
    echo "2  Suspended"
    echo "3  Deactivated"
    echo "5  Destroyed"
    echo

    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys?state=5" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key+json' | jq -r '[.resources | .[] 
         | {	name: .name, id: .id, state: .state, crn: .crn} ] 
         | [ .[] | with_entries( .key |= ascii_downcase ) ] | (.[0] 
         | keys_unsorted | @tsv), (.[]|.| map(.) | @tsv) ' | column -t -s $'\t'

	echo
	echo "Request complete."
	exit 0

}


#-----------------------------------------------------------------------------------------------------
# View a list of keys in a Key Protect instance
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
view_keys_list()
{
    echo
    echo "Listing keys for service ${service_name}..."
    echo
    echo "Values for State column:"
    echo "-------------------------"
    echo "0  Pre-activation"
    echo "1  Active"
    echo "2  Suspended"
    echo "3  Deactivated"
    echo "5  Destroyed"
    echo

    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key+json' | jq -r '[.resources | .[] 
         | {	name: .name, id: .id, state: .state, crn: .crn} ] 
         | [ .[] | with_entries( .key |= ascii_downcase ) ] | (.[0] 
         | keys_unsorted | @tsv), (.[]|.| map(.) | @tsv) ' | column -t -s $'\t'

	echo
	echo "Request complete."
	exit 0

}


#-----------------------------------------------------------------------------------------------------
# Import a key
#
# Arguments: key-type, key-name, key-payload
#    
#-----------------------------------------------------------------------------------------------------
import_key()
{

    display_usage() {
    
 		echo
		echo "USAGE:"
		echo "keyprotect.sh [service instance name] import-key [key-type] [key-name] [key-payload]"
		echo
		echo "Key Type"
		echo "-------------------------------------------------------------------------------------------"
		echo "standard     Creates a standard key.  Material from a standard key can be exported"
		echo "root         Creates a root key.  material from a root key can never be exported"
		echo
		echo "Note: Key material must be base64 encoded.  For a root key, the base64 decoded value must be"
		echo "      128, 192 or 256 bits.  This is 16 (128 bit), 24 (192 bit) or 32 (256 bit) Hex value"
		echo
		echo "      This command can create a 256 bit base64 encoded key for demo purposes:"
		echo
		echo "      openssl rand -base64 32"
		echo
		exit 1     
    }

	if [[ $# -lt 3 ]]; then

		# Key type is missing
		if [[ $# -lt 1 ]]; then
			echo
			echo "FAILED:  a key type is required"
			
			# there is a problem so display usage
			display_usage
		fi
	
		# Key name is missing
		if [[ $# -lt 2 ]]; then
		
			echo
			echo "FAILED: a key name is required"
			
			# there is a problem so display usage
			display_usage	
		fi


	
		# Key payload is missing
		if [[ $# -lt 3 ]]; then
			echo
			echo "FAILED: a key payload is required"
			# there is a problem so display usage
			display_usage
		fi

		# there is a problem so display usage
		display_usage
	fi

	# Get parameter values
	keyType="$1"
	keyName="$2"
	keyPayload="$3"

	# Validate the key type
	if [[ $keyType == "standard" || $keyType == "root" ]]; then
		echo
	else
		echo
		echo "FAILED:  key type: $keyType is invalid"
		display_usage
		exit 1
	fi


	# 
	if [[ $keyType == "root" ]]; then
	    isExtractable=false
	else
	    isExtractable=true
    fi
    
    echo "The value of isExtractable is ${isExtractable}"

	echo
	echo "Importing key ..."
	echo

	#set -x
	curl -s -X POST "https://${region}.kms.cloud.ibm.com/api/v2/keys" \
	  -H 'Authorization: Bearer '"$ibm_auth_token"'' \
	  -H 'bluemix-instance: '"$service_instance_guid"'' \
	  -H 'content-type: application/vnd.ibm.kms.key+json' \
	  -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.key+json",
				"collectionTotal": 1
			},
			"resources": [
				{
					"type": "application/vnd.ibm.kms.key+json",
					"name": "'"$keyName"'",
					"extractable": '"$isExtractable"',
					"payload": "'"$keyPayload"'"
				}
			]
		  }' | jq --color-output

	echo
	echo "Done."	
	echo
	echo "Request complete."
	exit 0

}


#-----------------------------------------------------------------------------------------------------
# Restore a key
#
# Arguments: key-id, key-payload
#    
#-----------------------------------------------------------------------------------------------------
restore_key()
{

    display_usage() {
    
 		echo
		echo "USAGE:"
		echo "keyprotect.sh [service instance name] restore-key [key-id] [key-payload]"
		echo
		echo "Note: The key-id can be found using the view-keys command.  You can only restore a key that"
		echo "      previously existed.  Deleted keys have a State of Destroyed.  You must wait 30 seconds"
		echo "      after deleting a key before it can be restored."
		echo
		exit 1   
    }

	
	if [[ $# -lt 2 ]]; then

		# Key id is missing
		if [[ $# -lt 1 ]]; then
			echo
			echo "FAILED:  a key id is required"
			
			# there is a problem so display usage
			display_usage
		fi
	
		# Key payload is missing
		if [[ $# -lt 2 ]]; then
			echo
			echo "FAILED: a key payload is required"
			# there is a problem so display usage
			display_usage
		fi

		# there is a problem so display usage
		display_usage
	fi

	# Get parameter values
	keyId="$1"
	keyPayload="$2"

	echo
	echo "Restoring key ..."
	echo

	set -x
	curl -s -X POST "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}?action=restore" \
	  -H 'Authorization: Bearer '"$ibm_auth_token"'' \
	  -H 'bluemix-instance: '"$service_instance_guid"'' \
	  -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.key+json",
				"collectionTotal": 1
			},
			"resources": [
				{
					"payload": "'"$keyPayload"'"
				}
			]
		  }' | jq --color-output

	echo
	echo "Done."	
	echo
	echo "Request complete."
	exit 0

}

#-----------------------------------------------------------------------------------------------------
# Enable Dual Authorization for a Key Protect Instance
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
enable_dual_auth()
{

    echo
    echo "Enabling Dual Authorization for service ${service_name}..."
    echo

    curl -s -X PUT "https://${region}.kms.cloud.ibm.com/api/v2/instance/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'Content-Type: application/vnd.ibm.kms.policy+json' \
      -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.policy+json",
    			"collectionTotal": 1
			},
    		"resources": [
				{
					"policy_type": "dualAuthDelete",
					"policy_data": {
						"enabled": true
					}
				}
    		]
  		  }'

	echo
	echo "Done."

	view_policies
	echo
	echo "Request complete."
	exit 0
}

#-----------------------------------------------------------------------------------------------------
# disable Dual Authorization for a Key Protect Instance
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
disable_dual_auth()
{

    echo
    echo "disabling Dual Authorization for service ${service_name}..."
    echo

    curl -s -X PUT "https://${region}.kms.cloud.ibm.com/api/v2/instance/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'Content-Type: application/vnd.ibm.kms.policy+json' \
      -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.policy+json",
    			"collectionTotal": 1
			},
    		"resources": [
				{
					"policy_type": "dualAuthDelete",
					"policy_data": {
						"enabled": false
					}
				}
    		]
  		  }'

	echo
	echo "Done."

	view_policies
	echo
	echo "Request complete."
	exit 0
}

#-----------------------------------------------------------------------------------------------------
# Set Allowed Networks policy for a Key Protect Instance
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
disable_public_endpoint()
{

    echo
    echo "Disabling public endpoint for service ${service_name}..."
    echo

    curl -s -X PUT "https://${region}.kms.cloud.ibm.com/api/v2/instance/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'Content-Type: application/vnd.ibm.kms.policy+json' \
      -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.policy+json",
    			"collectionTotal": 1
			},
    		"resources": [
				{
					"policy_type": "allowedNetwork",
					"policy_data": {
						"enabled": true,
						"attributes": {
							"allowed_network": "private-only"
						}
					}
				}
    		]
  		  }'

	echo
	echo "Done."

	view_policies
	echo
	echo "Request complete."
	exit 0
}

#-----------------------------------------------------------------------------------------------------
# Set Allowed Networks policy for a Key Protect Instance
#
# Arguments: none
#    
#-----------------------------------------------------------------------------------------------------
enable_public_endpoint()
{

    echo
    echo "enabling public endpoint for service ${service_name}..."
    echo

    curl -s -X PUT "https://${region}.kms.cloud.ibm.com/api/v2/instance/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'Content-Type: application/vnd.ibm.kms.policy+json' \
      -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.policy+json",
    			"collectionTotal": 1
			},
    		"resources": [
				{
					"policy_type": "allowedNetwork",
					"policy_data": {
						"enabled": true,
						"attributes": {
							"allowed_network": "public-and-private"
						}
					}
				}
    		]
  		  }'

	echo
	echo "Done."

	view_policies
	echo
	echo "Request complete."
	exit 0
}

#-----------------------------------------------------------------------------------------------------
# View the details for a key
#
# Arguments: key-id
#    
#-----------------------------------------------------------------------------------------------------
view_key()
{

	if [[ $# -lt 1 ]]; then
		echo
		echo "for view-key a key id is required"
		echo
		echo "USAGE:"
		echo "keyprotect.sh <key-protect-instance-name] view-key [key-id]"
		echo
		echo
		exit 1
	fi
	
	keyId=$1
	
	echo
	echo "viewing attributes for key ${keyId}..."
	echo
	
    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
	  -H 'accept: application/vnd.ibm.kms.policy+json' | jq --color-output
	
	echo
	echo "Request complete."
	exit 0
}

#-----------------------------------------------------------------------------------------------------
# View the material for a key  <-- I think this is redundant with view_key
#
# Arguments: key-id
#    
#-----------------------------------------------------------------------------------------------------
view_key_material()
{

	if [[ $# -lt 1 ]]; then
		echo
		echo "for view-key-material a key id is required"
		echo
		echo "USAGE:"
		echo "keyprotect.sh [service instance name] view-key-material [key-id]"
		echo
		echo
		exit 1
	fi
	
	keyId=$1
	
    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys/${key_id}" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
	  -H 'accept: application/vnd.ibm.kms.policy+json' | jq -r ' .resources[] | .payload '
	

	exit 0
}

#-----------------------------------------------------------------------------------------------------
# Set a key for deletion - the first step in Dual Deletion 
#
# Arguments: key-id
#    
#-----------------------------------------------------------------------------------------------------
set_key_deletion()
{

	if [[ $# -lt 1 ]]; then
		echo
		echo "for set-key-deletion a key id is required"
		echo
		echo "USAGE:"
		echo "keyprotect.sh [service instance name] set-key-deletion [key-id]"
		echo
		echo
		exit 1
	fi
	
	keyId=$1
	
    echo
    echo "Setting key ${keyId} in service ${service_name} for deletion..."
    echo

    curl -s -X POST "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}?action=setKeyForDeletion" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key_action+json' \
	  -H 'content-type: application/vnd.ibm.kms.key_action+json' \
      -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.policy+json",
    			"collectionTotal": 1
			},
    		"resources": [
				{
					"policy_type": "dualAuthDelete",
					"policy_data": {
						"enabled": true
					}
				}
    		]
  		  }' | jq --color-output

	echo
	echo "Done."	
	echo
	echo "viewing policies for key ${keyId}..."
	echo
	
    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
	  -H 'accept: application/vnd.ibm.kms.policy+json' | jq --color-output
	
	echo
	echo "Request complete."
	exit 0
}


#-----------------------------------------------------------------------------------------------------
# Unet a key for deletion - Use this function to "undo" a "set key for deletion" 
#
# Arguments: key-id
#    
#-----------------------------------------------------------------------------------------------------
unset_key_deletion()
{

	if [[ $# -lt 1 ]]; then
		echo
		echo "for set-key-deletion a key id is required"
		echo
		echo "USAGE:"
		echo "keyprotect.sh [service instance name] set-key-deletion [key-id]"
		echo
		echo
		exit 1
	fi
	
	keyId=$1
	
    echo
    echo "Unsetting key ${keyId} in service ${service_name} for deletion..."
    echo

    curl -s -X POST "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}?action=unsetKeyForDeletion" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
      -H 'accept: application/vnd.ibm.kms.key_action+json' \
	  -H 'content-type: application/vnd.ibm.kms.key_action+json' \
      -d '{
			"metadata": {
				"collectionType": "application/vnd.ibm.kms.policy+json",
    			"collectionTotal": 1
			},
    		"resources": [
				{
					"policy_type": "dualAuthDelete",
					"policy_data": {
						"enabled": true
					}
				}
    		]
  		  }' | jq --color-output

	echo
	echo "Done."	
	echo
	echo "viewing policies for key ${keyId}..."
	echo
	
    curl -s "https://${region}.kms.cloud.ibm.com/api/v2/keys/${keyId}/policies" \
      -H 'Authorization: Bearer '"$ibm_auth_token"'' \
      -H 'bluemix-instance: '"$service_instance_guid"'' \
	  -H 'accept: application/vnd.ibm.kms.policy+json' | jq --color-output
	
	echo
	echo "Request complete."
	exit 0
}


#-----------------------------------------------------------------------------------------------------
# Main script execution starts here
#-----------------------------------------------------------------------------------------------------

service_name="$1"
kp_command="$2"
key_id="$3"
# Note: original way was to always have key-id as 3rd arg.  New way is to move command logic to functions and
#       pass in argument 3 and higher in raw form directly to the function and let it decide what they mean.
#       For now, key_id is still set to $3 so that logic outside of functions still works.

# Always need to arguments:  service-instance-name and a command
if [[ $# -lt 2  ]]; then
	display_usage
    exit 1
fi

# Only need to do these commands if a service name is present
if [[ $# -ge 1 ]]; then

	# Get the service instance details for $service_name
	service_instance_details=$(ibmcloud resource service-instance $service_name --output JSON)
	
	

	# Get the GUID for the specified Key Protect instance
	service_instance_guid=$(echo $service_instance_details| jq -r '.[0].guid')
#	service_instance_guid=$(ibmcloud resource service-instance $service_name --output JSON | jq -r '.[0].guid')

	# Get the region for the Key Protect Instance
#	region=$(echo $service_instance_details | jq -r '.[0].region_id')

#   Note:  the line below sets the API endpoint to the private network.
#   To use the public network comment the line below and uncomment the one above.
	region=private.$(echo $service_instance_details | jq -r '.[0].region_id')


	# Get the bearer token to use for authentication.
	# Note:  This assumes that the user has previously logged into the IBM Cloud CLI
	ibm_auth_token=$(ibmcloud iam oauth-tokens | awk '{ print $4}')
fi


#-----------------------------------------------------------------------------------------------------
# View the policies for the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [ $kp_command == "help" ] || [ $kp_command == "h" ]; then
	display_usage
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View the policies for the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-policies" ]]; then
	view_policies
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View the policies for a key in the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-key-policies" ]]; then
	view_key_policies $3
fi


#-----------------------------------------------------------------------------------------------------
# View the keys for the specified Key Protect instance in JSON format
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-keys" ]]; then
	view_keys
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View the keys for the specified Key Protect instance in JSON format
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-deleted-keys" ]]; then
	view_deleted_keys
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View the deleted keys for the specified Key Protect instance in List format
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-deleted-keys-list" ]]; then
	view_deleted_keys_list
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View the keys for the specified Key Protect instance in List format
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-keys-list" ]]; then
	view_keys_list
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# View the attributes for a key in the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-key" ]]; then
	view_key $3
fi

#-----------------------------------------------------------------------------------------------------
# Import a key into the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "import-key" ]]; then
	import_key $3 $4 $5
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# Import a key into the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "restore-key" ]]; then
	restore_key $3 $4
	exit 0
fi

#-----------------------------------------------------------------------------------------------------
# Enable dual authorization delete policy for the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "enable-dual-auth" ]]; then
	enable_dual_auth
fi

#-----------------------------------------------------------------------------------------------------
# Disable dual authorization delete policy for the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "disable-dual-auth" ]]; then
	disable_dual_auth
fi

#-----------------------------------------------------------------------------------------------------
# Disable public network for the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "disable-public-endpoint" ]]; then
	disable_public_endpoint
fi

#-----------------------------------------------------------------------------------------------------
# enable public network for the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "enable-public-endpoint" ]]; then
	enable_public_endpoint
fi

#-----------------------------------------------------------------------------------------------------
# View the attributes for a key in the specified Key Protect instance
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "view-key-material" ]]; then
	view_key_material $3
fi

#-----------------------------------------------------------------------------------------------------
# Set the specified key for deletion (first auth)
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "set-key-deletion" ]]; then
	set_key_deletion $3
fi

#-----------------------------------------------------------------------------------------------------
# Unset the deletion (first auth) of the specified Key
#-----------------------------------------------------------------------------------------------------
if [[ $kp_command == "unset-key-deletion" ]]; then
	unset_key_deletion $3
fi

#-----------------------------------------------------------------------------------------------------



echo
echo "Error: Invalid command"
echo
display_usage




```

