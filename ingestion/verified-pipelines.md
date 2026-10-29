# Verified pipeline definitions and scope review

[Back to project](../README.md)

Reviewed from six owner-supplied Fabric pipeline ZIP exports on 29 October 2026. Each template can bundle child pipeline resources. Only the named root resource is shown under each heading, avoiding duplicate embedded definitions. Connection endpoints, workspace/item IDs and modification metadata have been removed or replaced with placeholders. These are sanitised configuration walkthroughs, not unchanged import-ready templates.

## Confirmed correction needed: Gold fact scope

The processing parent passes both scope parameters correctly to both children. Silver maps them into SQL parameters. However, the Gold fact activity currently supplies literal empty strings for `ForceId` and `SourceMonth`. The wrapper normalises those to NULL, so Gold performs a full fact replacement even when Silver is scoped. This explains the supplied audit evidence.

In `PL_Police_Load_Gold_StopSearch`, select `SP_Load_FactStopSearch` and replace the two empty procedure parameter values with dynamic content:

**ForceId**

```text
@if(empty(trim(pipeline().parameters.p_force_id)), null, trim(pipeline().parameters.p_force_id))
```

**SourceMonth**

```text
@if(empty(trim(pipeline().parameters.p_source_month)), null, trim(pipeline().parameters.p_source_month))
```

Retain `@pipeline().RunId` for `PipelineRunId`, save the Gold pipeline and run the processing parent with Metropolitan / 2025-08. A fresh Gold audit should then show that scope and 14,899 rows, assuming unchanged source data. The correction is proposed here; Fabric has not been modified by this repository update.

## Export interpretation

- Backfill Lookup currently reads `control.stopsearchretryplan`, and the ForEach is sequential.
- Reference pipeline still downloads crime categories for every available month before calling the reference notebook. These are exploratory dependencies, not part of the final analytical scope.
- REST Copy uses GET, a 100-second request timeout and JSON `arrayOfObjects` output. Activity retry policies shown in the exports are 0; the prior four-request recovery was manual, not automatic retry logic.
- Processing and child defaults are Metropolitan / 2025-08 in these exports, not empty strings. Blank input remains a supported explicit full-scope request through the Silver/Gold normalisation logic.
- Invoke activities are the newer `InvokePipeline` type with completion waiting; earlier setup instructions suggested Legacy, but the actual export is authoritative.
- Exported stored-procedure name strings contain an extra leading bracket (`[[...`). Successful execution was separately observed. This review preserves that export text rather than assuming a runtime failure or silently rewriting it. Verify the selected procedure through the activity UI when reproducing.

## PL_Police_Load_All_Stop_Search

```json
{
  "name": "PL_Police_Load_All_Stop_Search",
  "type": "pipelines",
  "apiVersion": "2018-06-01",
  "properties": {
    "activities": [
      {
        "name": "Lookup_Stop_Search_Load_Plan",
        "type": "Lookup",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "source": {
            "type": "LakehouseTableSource"
          },
          "firstRowOnly": false,
          "datasetSettings": {
            "annotations": [],
            "linkedService": {
              "name": "LH_Uk_Police_Crime",
              "properties": {
                "annotations": [],
                "type": "Lakehouse",
                "typeProperties": {
                  "workspaceId": "<ENVIRONMENT_ID>",
                  "artifactId": "[parameters('LH_Uk_Police_Crime')]",
                  "rootFolder": "Tables"
                }
              }
            },
            "type": "LakehouseTable",
            "schema": [],
            "typeProperties": {
              "schema": "control",
              "table": "stopsearchretryplan"
            }
          }
        }
      },
      {
        "name": "ForEach_Stop_Search_Request",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "Lookup_Stop_Search_Load_Plan",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "typeProperties": {
          "items": {
            "value": "@activity('Lookup_Stop_Search_Load_Plan').output.value",
            "type": "Expression"
          },
          "isSequential": true,
          "activities": [
            {
              "name": "Invoke_Stop_Search_Load",
              "type": "InvokePipeline",
              "dependsOn": [],
              "policy": {
                "timeout": "0.12:00:00",
                "retry": 0,
                "retryIntervalInSeconds": 30,
                "secureOutput": false,
                "secureInput": false
              },
              "typeProperties": {
                "waitOnCompletion": true,
                "operationType": "InvokeFabricPipeline",
                "pipelineId": "<ENVIRONMENT_ID>",
                "workspaceId": "<ENVIRONMENT_ID>",
                "parameters": {
                  "p_force": {
                    "value": "@item().force_id",
                    "type": "Expression"
                  },
                  "p_month": {
                    "value": "@item().source_month",
                    "type": "Expression"
                  }
                }
              },
              "externalReferences": {
                "connection": "[parameters('FabricPipelineConnection')]"
              }
            }
          ]
        }
      }
    ]
  },
  "dependsOn": []
}
```

## PL_Police_Load_Forces

```json
{
  "name": "PL_Police_Load_Forces",
  "type": "pipelines",
  "apiVersion": "2018-06-01",
  "properties": {
    "activities": [
      {
        "name": "Copy_Forces_API_To_Raw",
        "type": "Copy",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "source": {
            "type": "RestSource",
            "httpRequestTimeout": "00:01:40",
            "requestInterval": "00.00:00:00.010",
            "requestMethod": "GET",
            "datasetSettings": {
              "annotations": [],
              "type": "RestResource",
              "typeProperties": {
                "relativeUrl": "forces"
              },
              "schema": [],
              "externalReferences": {
                "connection": "[parameters('PoliceRestConnection')]"
              }
            }
          },
          "sink": {
            "type": "JsonSink",
            "storeSettings": {
              "type": "AzureBlobStorageWriteSettings"
            },
            "formatSettings": {
              "type": "JsonWriteSettings",
              "filePattern": "arrayOfObjects"
            },
            "datasetSettings": {
              "annotations": [],
              "linkedService": {
                "name": "LH_Uk_Police_Crime",
                "properties": {
                  "annotations": [],
                  "type": "Lakehouse",
                  "typeProperties": {
                    "workspaceId": "<ENVIRONMENT_ID>",
                    "artifactId": "[parameters('LH_Uk_Police_Crime')]",
                    "rootFolder": "Files"
                  }
                }
              },
              "type": "Json",
              "typeProperties": {
                "location": {
                  "type": "LakehouseLocation",
                  "fileName": "forces.json",
                  "folderPath": "raw/police/forces"
                }
              },
              "schema": {}
            }
          },
          "enableStaging": false
        }
      },
      {
        "name": "Copy_Available_Months_API_To_Raw",
        "type": "Copy",
        "dependsOn": [
          {
            "activity": "Copy_Forces_API_To_Raw",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "source": {
            "type": "RestSource",
            "httpRequestTimeout": "00:01:40",
            "requestInterval": "00.00:00:00.010",
            "requestMethod": "GET",
            "datasetSettings": {
              "annotations": [],
              "type": "RestResource",
              "typeProperties": {
                "relativeUrl": "crimes-street-dates"
              },
              "schema": [],
              "externalReferences": {
                "connection": "[parameters('PoliceRestConnection')]"
              }
            }
          },
          "sink": {
            "type": "JsonSink",
            "storeSettings": {
              "type": "AzureBlobStorageWriteSettings"
            },
            "formatSettings": {
              "type": "JsonWriteSettings",
              "filePattern": "arrayOfObjects"
            },
            "datasetSettings": {
              "annotations": [],
              "linkedService": {
                "name": "LH_Uk_Police_Crime",
                "properties": {
                  "annotations": [],
                  "type": "Lakehouse",
                  "typeProperties": {
                    "workspaceId": "<ENVIRONMENT_ID>",
                    "artifactId": "[parameters('LH_Uk_Police_Crime')]",
                    "rootFolder": "Files"
                  }
                }
              },
              "type": "Json",
              "typeProperties": {
                "location": {
                  "type": "LakehouseLocation",
                  "fileName": "available_months.json",
                  "folderPath": "raw/police/available_months"
                }
              },
              "schema": {}
            }
          },
          "enableStaging": false
        }
      },
      {
        "name": "Lookup_Available_Months",
        "type": "Lookup",
        "dependsOn": [
          {
            "activity": "Copy_Available_Months_API_To_Raw",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "source": {
            "type": "JsonSource",
            "storeSettings": {
              "type": "LakehouseReadSettings",
              "recursive": true,
              "enablePartitionDiscovery": false
            },
            "formatSettings": {
              "type": "JsonReadSettings"
            }
          },
          "firstRowOnly": false,
          "datasetSettings": {
            "annotations": [],
            "linkedService": {
              "name": "LH_Uk_Police_Crime",
              "properties": {
                "annotations": [],
                "type": "Lakehouse",
                "typeProperties": {
                  "workspaceId": "<ENVIRONMENT_ID>",
                  "artifactId": "[parameters('LH_Uk_Police_Crime')]",
                  "rootFolder": "Files"
                }
              }
            },
            "type": "Json",
            "typeProperties": {
              "location": {
                "type": "LakehouseLocation",
                "fileName": "available_months.json",
                "folderPath": "raw/police/available_months"
              }
            },
            "schema": {}
          }
        }
      },
      {
        "name": "ForEach_Available_Month",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "Lookup_Available_Months",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "typeProperties": {
          "items": {
            "value": "@activity('Lookup_Available_Months').output.value",
            "type": "Expression"
          },
          "isSequential": true,
          "activities": [
            {
              "name": "Copy_Crime_Categories_By_Month",
              "type": "Copy",
              "dependsOn": [],
              "policy": {
                "timeout": "0.12:00:00",
                "retry": 0,
                "retryIntervalInSeconds": 30,
                "secureOutput": false,
                "secureInput": false
              },
              "typeProperties": {
                "source": {
                  "type": "RestSource",
                  "httpRequestTimeout": "00:01:40",
                  "requestInterval": "00.00:00:00.010",
                  "requestMethod": "GET",
                  "datasetSettings": {
                    "annotations": [],
                    "type": "RestResource",
                    "typeProperties": {
                      "relativeUrl": {
                        "value": "@concat('crime-categories?date=', item().date)",
                        "type": "Expression"
                      }
                    },
                    "schema": [],
                    "externalReferences": {
                      "connection": "[parameters('PoliceRestConnection')]"
                    }
                  }
                },
                "sink": {
                  "type": "JsonSink",
                  "storeSettings": {
                    "type": "LakehouseWriteSettings"
                  },
                  "formatSettings": {
                    "type": "JsonWriteSettings",
                    "filePattern": "arrayOfObjects"
                  },
                  "datasetSettings": {
                    "annotations": [],
                    "linkedService": {
                      "name": "LH_Uk_Police_Crime",
                      "properties": {
                        "annotations": [],
                        "type": "Lakehouse",
                        "typeProperties": {
                          "workspaceId": "<ENVIRONMENT_ID>",
                          "artifactId": "[parameters('LH_Uk_Police_Crime')]",
                          "rootFolder": "Files"
                        }
                      }
                    },
                    "type": "Json",
                    "typeProperties": {
                      "location": {
                        "type": "LakehouseLocation",
                        "fileName": "crime_categories.json",
                        "folderPath": {
                          "value": "@concat('raw/police/crime_categories/month=', item().date)",
                          "type": "Expression"
                        }
                      }
                    },
                    "schema": {}
                  }
                },
                "enableStaging": false
              }
            }
          ]
        }
      },
      {
        "name": "Load_Bronze_Reference_Tables",
        "type": "TridentNotebook",
        "dependsOn": [
          {
            "activity": "ForEach_Available_Month",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "notebookId": "<ENVIRONMENT_ID>",
          "workspaceId": "<ENVIRONMENT_ID>"
        }
      }
    ]
  },
  "dependsOn": []
}
```

## PL_Police_Load_Gold_StopSearch

```json
{
  "name": "PL_Police_Load_Gold_StopSearch",
  "type": "pipelines",
  "apiVersion": "2018-06-01",
  "properties": {
    "activities": [
      {
        "name": "SP_Load_DimDate",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "storedProcedureName": "[[gold].[usp_LoadDimDate]"
        },
        "linkedService": {
          "name": "WH_Police_Analytics",
          "properties": {
            "annotations": [],
            "type": "DataWarehouse",
            "typeProperties": {
              "endpoint": "<WAREHOUSE_SQL_ENDPOINT>",
              "artifactId": "[parameters('WH_Police_Analytics')]",
              "workspaceId": "<ENVIRONMENT_ID>"
            }
          }
        }
      },
      {
        "name": "SP_Load_DimForce",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [
          {
            "activity": "SP_Load_DimDate",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "storedProcedureName": "[[gold].[usp_LoadDimForce]"
        },
        "linkedService": {
          "name": "WH_Police_Analytics",
          "properties": {
            "annotations": [],
            "type": "DataWarehouse",
            "typeProperties": {
              "endpoint": "<WAREHOUSE_SQL_ENDPOINT>",
              "artifactId": "[parameters('WH_Police_Analytics')]",
              "workspaceId": "<ENVIRONMENT_ID>"
            }
          }
        }
      },
      {
        "name": "SP_Load_ProfileDimensions",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [
          {
            "activity": "SP_Load_DimForce",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "storedProcedureName": "[[gold].[usp_LoadProfileDimensions]"
        },
        "linkedService": {
          "name": "WH_Police_Analytics",
          "properties": {
            "annotations": [],
            "type": "DataWarehouse",
            "typeProperties": {
              "endpoint": "<WAREHOUSE_SQL_ENDPOINT>",
              "artifactId": "[parameters('WH_Police_Analytics')]",
              "workspaceId": "<ENVIRONMENT_ID>"
            }
          }
        }
      },
      {
        "name": "SP_Load_FactStopSearch",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [
          {
            "activity": "SP_Load_ProfileDimensions",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "storedProcedureName": "[[gold].[usp_RunFactStopSearchLoad]",
          "storedProcedureParameters": {
            "ForceId": {
              "value": "",
              "type": "String"
            },
            "PipelineRunId": {
              "value": {
                "value": "@pipeline().RunId",
                "type": "Expression"
              },
              "type": "String"
            },
            "SourceMonth": {
              "value": "",
              "type": "String"
            }
          }
        },
        "linkedService": {
          "name": "WH_Police_Analytics",
          "properties": {
            "annotations": [],
            "type": "DataWarehouse",
            "typeProperties": {
              "endpoint": "<WAREHOUSE_SQL_ENDPOINT>",
              "artifactId": "[parameters('WH_Police_Analytics')]",
              "workspaceId": "<ENVIRONMENT_ID>"
            }
          }
        }
      }
    ],
    "parameters": {
      "p_force_id": {
        "type": "string",
        "defaultValue": "metropolitan"
      },
      "p_source_month": {
        "type": "string",
        "defaultValue": "2025-08"
      }
    }
  },
  "dependsOn": []
}
```

## PL_Police_Load_Silver_StopSearch

```json
{
  "name": "PL_Police_Load_Silver_StopSearch",
  "type": "pipelines",
  "apiVersion": "2018-06-01",
  "properties": {
    "activities": [
      {
        "name": "SP_Load_Silver_StopSearch",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "storedProcedureName": "[[silver].[usp_RunStopSearchLoad]",
          "storedProcedureParameters": {
            "ForceId": {
              "value": {
                "value": "@if(empty(trim(pipeline().parameters.p_force_id)), null, trim(pipeline().parameters.p_force_id))",
                "type": "Expression"
              },
              "type": "String"
            },
            "PipelineRunId": {
              "value": {
                "value": "@pipeline().RunId",
                "type": "Expression"
              },
              "type": "String"
            },
            "SourceMonth": {
              "value": {
                "value": "@if(empty(trim(pipeline().parameters.p_source_month)), null, trim(pipeline().parameters.p_source_month))",
                "type": "Expression"
              },
              "type": "String"
            }
          }
        },
        "linkedService": {
          "name": "WH_Police_Analytics",
          "properties": {
            "annotations": [],
            "type": "DataWarehouse",
            "typeProperties": {
              "endpoint": "<WAREHOUSE_SQL_ENDPOINT>",
              "artifactId": "[parameters('WH_Police_Analytics')]",
              "workspaceId": "<ENVIRONMENT_ID>"
            }
          }
        }
      }
    ],
    "parameters": {
      "p_force_id": {
        "type": "string",
        "defaultValue": "metropolitan"
      },
      "p_source_month": {
        "type": "string",
        "defaultValue": "2025-08"
      }
    }
  },
  "dependsOn": []
}
```

## PL_Police_Load_Stop_Search

```json
{
  "name": "PL_Police_Load_Stop_Search",
  "type": "pipelines",
  "apiVersion": "2018-06-01",
  "properties": {
    "activities": [
      {
        "name": "Copy_Stop_Search_API_To_Raw",
        "type": "Copy",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "source": {
            "type": "RestSource",
            "httpRequestTimeout": "00:01:40",
            "requestInterval": "00.00:00:00.010",
            "requestMethod": "GET",
            "datasetSettings": {
              "annotations": [],
              "type": "RestResource",
              "typeProperties": {
                "relativeUrl": {
                  "value": "@concat('stops-force?force=', pipeline().parameters.p_force, '&date=', pipeline().parameters.p_month)",
                  "type": "Expression"
                }
              },
              "schema": [],
              "externalReferences": {
                "connection": "[parameters('PoliceRestConnection')]"
              }
            }
          },
          "sink": {
            "type": "JsonSink",
            "storeSettings": {
              "type": "AzureBlobStorageWriteSettings"
            },
            "formatSettings": {
              "type": "JsonWriteSettings",
              "filePattern": "arrayOfObjects"
            },
            "datasetSettings": {
              "annotations": [],
              "linkedService": {
                "name": "LH_Uk_Police_Crime",
                "properties": {
                  "annotations": [],
                  "type": "Lakehouse",
                  "typeProperties": {
                    "workspaceId": "<ENVIRONMENT_ID>",
                    "artifactId": "[parameters('LH_Uk_Police_Crime')]",
                    "rootFolder": "Files"
                  }
                }
              },
              "type": "Json",
              "typeProperties": {
                "location": {
                  "type": "LakehouseLocation",
                  "fileName": "stop_search.json",
                  "folderPath": {
                    "value": "@concat('raw/police/stop_search/force=', pipeline().parameters.p_force, '/month=', pipeline().parameters.p_month)",
                    "type": "Expression"
                  }
                }
              },
              "schema": {}
            }
          },
          "enableStaging": false
        }
      }
    ],
    "parameters": {
      "p_force": {
        "type": "string",
        "defaultValue": "northumbria"
      },
      "p_month": {
        "type": "string",
        "defaultValue": "2026-08"
      }
    }
  },
  "dependsOn": []
}
```

## PL_Police_Process_StopSearch

```json
{
  "name": "PL_Police_Process_StopSearch",
  "type": "pipelines",
  "apiVersion": "2018-06-01",
  "properties": {
    "activities": [
      {
        "name": "Invoke_Load_Silver",
        "type": "InvokePipeline",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "waitOnCompletion": true,
          "operationType": "InvokeFabricPipeline",
          "pipelineId": "<ENVIRONMENT_ID>",
          "workspaceId": "<ENVIRONMENT_ID>",
          "parameters": {
            "p_force_id": {
              "value": "@pipeline().parameters.p_force_id",
              "type": "Expression"
            },
            "p_source_month": {
              "value": "@pipeline().parameters.p_source_month",
              "type": "Expression"
            }
          }
        },
        "externalReferences": {
          "connection": "[parameters('FabricPipelineConnection')]"
        }
      },
      {
        "name": "Invoke_Load_Gold",
        "type": "InvokePipeline",
        "dependsOn": [
          {
            "activity": "Invoke_Load_Silver",
            "dependencyConditions": [
              "Succeeded"
            ]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 0,
          "retryIntervalInSeconds": 30,
          "secureOutput": false,
          "secureInput": false
        },
        "typeProperties": {
          "waitOnCompletion": true,
          "operationType": "InvokeFabricPipeline",
          "pipelineId": "<ENVIRONMENT_ID>",
          "workspaceId": "<ENVIRONMENT_ID>",
          "parameters": {
            "p_force_id": {
              "value": "@pipeline().parameters.p_force_id",
              "type": "Expression"
            },
            "p_source_month": {
              "value": "@pipeline().parameters.p_source_month",
              "type": "Expression"
            }
          }
        },
        "externalReferences": {
          "connection": "[parameters('FabricPipelineConnection')]"
        }
      }
    ],
    "parameters": {
      "p_force_id": {
        "type": "string",
        "defaultValue": "metropolitan"
      },
      "p_source_month": {
        "type": "string",
        "defaultValue": "2025-08"
      }
    }
  },
  "dependsOn": []
}
```

