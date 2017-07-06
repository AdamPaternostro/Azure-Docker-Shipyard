# Azure-Docker-Shipyard
Demo of using Docker contatiners in Azure Batch (using Shipyard).  This also shows how to mount a Docker Volume using Shipyard.

## Links
- https://github.com/Azure/batch-shipyard
- https://azure.github.io/batch-shipyard/

## Create Azure Resource Group
- Create a Resource Group (e.g. AdamShipyardDemo)

## Create Azure Batch Service (create in the resource group above)
- Create a "Batch Service" account (e.g. adamshipyardbatchservice)
- When creating the "Batch Service" create a storage account (e.g. adamshipyardstorage)

## Create a Linux VM (create in the resource group above)
- Create Ubuntu Server 16.04 LTS  (e.g. adamshipyardvm)
    Username: shipyarduser  Password: <<REMOVED>> 
    Hard disk: HDD
    Size: D1_V2 (does not need to be powerful)
    Use Managed Disk: Yes
    Monitoring: Disabled
    Use the defualts for Networking

## Install Docker and Shipyard
```
sudo apt-get -y install apt-transport-https ca-certificates curl
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu  $(lsb_release -cs) stable"  
sudo apt-get update
sudo apt-get -y install docker-ce
```

### Test docker
```
sudo docker run hello-world
```

### For Shipyard image
```
sudo docker pull alfpark/batch-shipyard:cli-latest 
```

## Create a Docker program
- Create a directory mkdir docker
- Create a file named Dockerfile and place this in the file
```
FROM alpine
WORKDIR /app
RUN apk add --update bash
RUN apk add --update curl && rm -rf /var/cache/apk/*
ADD download.sh /app
CMD ["bash","./download.sh"]
```

- Create a file named download.sh and place this in the file
```
#!/bin/bash
for i in {1..10}
do
   filename=$(date +"%m-%d-%y-%T").pdf
   echo "Starting Download: $filename"
   curl -o /share/$filename http://ipv4.download.thinkbroadband.com/20MB.zip
   echo "Downloaded: $filename"
done
```

- Build the Docker image
```
sudo docker build -t adamshipyarddockerimage .
```

- List the images
```
sudo docker images
```

- Run the image locally
```
sudo docker run -v ~/mnt/share:/share adamshipyarddockerimage 
```

- Create a respository on Dockerhub (e.g. adamshipyardrepository)

- Upload image to repository
```
sudo docker login
sudo docker tag adamshipyarddockerimage adampaternostro/adamshipyarddockerimage:latest
sudo docker push adampaternostro/adamshipyarddockerimage:latest
```

## Create Shipyard files
- Go back to your home directory (cd ..)
- Make a directory (e.g. config)
- cd config

- Create a file named "config.json"
  - Description: This contains a reference to a storage account used by Shipyard for its own internal purposes.  It also has a reference to our Docker image as well a Data Volume so we know where to download our files.
  - For full schema: https://github.com/Azure/batch-shipyard/blob/master/docs/12-batch-shipyard-configuration-global.md
```
  {  
     "batch_shipyard":{  
        "storage_account_settings":"mystorageaccount"
     },
     "global_resources":{  
        "docker_images":[  
           "adampaternostro/adamshipyarddockerimage:latest"
        ],
        "docker_volumes":{  
           "data_volumes":{  
              "ephemeraldisk":{  
                 "host_path":"/mnt/docker-tmp",
                 "container_path":"/share"
              }
           }
        }
     }
  }
```

- Create a file named "credentials.json"
  - For full schema: https://github.com/Azure/batch-shipyard/blob/master/docs/11-batch-shipyard-configuration-credentials.md
  - Change the Batch: account_key and account_service_url
  - Change the mystorageaccount: account and account key (NOTE: keep the name mystorageaccount since it is referenced in the "config.json")
```
{  
   "credentials":{  
      "batch":{  
         "account_key":"<<REMOVED>>",
         "account_service_url":"https://adamshipyardbatchservice.eastus2.batch.azure.com"
      },
      "storage":{  
         "mystorageaccount":{  
            "account":"adamshipyardstorage",
            "account_key":"<<REMOVED>>",
            "endpoint":"core.windows.net"
         }
      }
   }
}
```

- Create a file named "jobs.json"
  - For full schema: https://github.com/Azure/batch-shipyard/blob/master/docs/14-batch-shipyard-configuration-jobs.md
```
{  
   "job_specifications":[  
      {  
         "id":"adamshipyardjob",
         "data_volumes":[  
            "ephemeraldisk"
         ],
         "tasks":[  
            {  
               "image":"adampaternostro/adamshipyarddockerimage:latest",
               "remove_container_after_exit":true,
               "command":"bash /app/download.sh"
            }
         ]
      }
   ]
}
```

- Create a file named "pool.json"
  - For full schema: https://github.com/Azure/batch-shipyard/blob/master/docs/13-batch-shipyard-configuration-pool.md
```
{  
   "pool_specification":{  
      "id":"adampool",
      "vm_size":"STANDARD_D1_V2",
      "vm_count":{  
         "dedicated":2,
         "low_priority":0
      },
      "vm_configuration":{  
         "platform_image":{  
            "publisher":"Canonical",
            "offer":"UbuntuServer",
            "sku":"16.04-LTS"
         }
      },
      "ssh":{  
         "username":"docker"
      },
      "reboot_on_start_task_failed":false,
      "block_until_all_global_resources_loaded":true
   }
}
```

## Run Shipyard
1 - Create the batch pool to run our Docker container on
```
sudo docker run --rm -it -v /home/shipyarduser/config:/configs -e SHIPYARD_CONFIGDIR=/configs alfpark/batch-shipyard:cli-latest pool add
```

2 - Run the Docker container (you can use stdout.txt or stderr.txt)
```
sudo docker run --rm -it -v /home/shipyarduser/config:/configs -e SHIPYARD_CONFIGDIR=/configs alfpark/batch-shipyard:cli-latest jobs add --tail stdout.txt
```

3 - Delete our pool (you do not have to do this, you can set the auto scaling down to zero when the pool is not in use)
```
sudo docker run --rm -it -v /home/shipyarduser/config:/configs -e SHIPYARD_CONFIGDIR=/configs alfpark/batch-shipyard:cli-latest pool del
```




