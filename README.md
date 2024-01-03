# Commission calculation - 

## Introduction
This document serves as a development guide and proposal for an automated system designed to manage and calculate sales commissions for BMW's sales agents. The purpose of this system is to simplify the current commission calculation process, reduce potential errors, and streamline the overall management of sales data.

The proposed system will automate tasks that are traditionally manual and time-consuming, such as updating and maintaining commission matrices and agent sales targets. It will leverage cloud technologies to ensure data is managed efficiently and securely. Additionally, the system is designed to integrate smoothly with BMW's existing data infrastructure, including the Midas API and CoreView platform, to facilitate seamless data flow and access.

The following sections will outline the detailed requirements, tools, and architecture for the system. They will provide insight into the functionalities it will offer, the technology stack it will use, the structure of its various components and a time estimation for the development process. Our goal is to provide BMW with a development plan for a reliable and efficient solution for commission management.

## Project requirements

### Functionality
- Upload Excel files containing Commission Matrices to AWS. This data should be easily accessible in order to do commission calculations. Should support overwriting of Matrices.
- Upload Excel files containing Agent Volume Target Sales data. Should support overwriting of agent sales targets.
- Montly read sales data from Midas API for the previous month, calculate the commission earned by agents, and write this data to CoreView.
- Montly read the clawback list, see if any of the contracts were cancelled within 6 months. If it was cancelled within 6 months, the commission that was paid for that contract should be nullified.

### Tools
- Github Actions for CI/CD (requirement BWM).
- Terraform for infrastructure (requirement BMW).
- Python will be used for lambda development since this project is mostly about data processing, which Python excels at.

### Throughput needs
- The files to be uploaded are relatively small. Lambda will be used for parsing.
- The data to be read and processed monthly from Midas is not very substantial. Lambda will be used for this as well.

### AWS components
AWS S3, Lambda, DynamoDB will cover the scalability and throughput needs of the product.

### Commission matrix file structure
This is how the Matrix data will be structured in Excel.

![Commision Matrix Data](./data_formats/matrix_dummy_data.png)

The matrix placement in the excel file should follow a standardised layout. A proposition is that the first matrix should be placed in the top left corner, and every new matrix placed to the right of the previous matrix, with a blank column in between. Another option is a new matrix on every "page" of the excel document. The files should be uploaded in .xlsx format. The .xlsx files should follow a predetermined naming convention, in order to uniquely identify that the file is a Matrix file, and which year the matrices should apply to. I suggest matrix_yyyy.xlsx, but anything that follows the previously mentioned prerequisites should suffice.

### Sales volume targets file structure
This is how the Agent Sales Target Volumes data will look.

![Volume Targets Data](./data_formats/volume_targets_dummy_data.png)

This looks pretty straight forward from a data processing perspective. The files should be uploaded in .xlsx format. As with the Matrix files, these files should follow a predetermined naming convention, in order to uniquely identify that the file is a Sales Target file, and which year it represents. I suggest sales_targets_yyyy.xlsx.

### Data needed from Midas

#### Commission calculation
To do the commission calculations the following data is needed from Midas:
- The total number of financed contracts per agent per market (total_number_of_financed_contracts).
- The total number of contracts per agent per market (total_number_of_contracts).
- Total financed sales volume per agent per market (financed_sales_volume_by_agent).

This is how the penetration rate is calculated:

```
penetration_rate = total_number_of_financed_contracts / total_number_of_contracts
```

This is how the target volume achieved is calculated:
```
target_volume_achieved = financed_sales_volume_by_agent / target_financed_sales_volume_by_agent
```

The rest of the data required for the commission calculation is pulled from DynamoDB (commission matrix and target financed sales volumes).

#### Clawback calculation
A clawback list will be fetched from Midas. The clawback lists contain a list of terminated contracts for the previous month. This list will be used to determine if there should be any clawbacks on previously issued commissions. 

### Midas API 
**TODO - Waiting for documentation.**

### Core View integration
**TODO - Waiting for documentation.**

## Architecture

### Overview
Event-driven serverless architecture will be used for all functionality. The functionality can be divided into two parts. The first part (blue) involves uploading and parsing data. Here S3 will be used to upload and store Matrix and Sales Target files. Upon upload a lambda will parse the data in these files into DynamoDB according to the database schema. The second part (green) involves reading data from an external AWS account, performing calculations on this data, and write the result to CoreView.

![Initial draft of architecture](architecture.svg)

### Calculation lambda
The calculation lambda should be triggered via a Cronjob at the first date of every month.

This should be done for every agent:
1. From Midas, retrieve all contracts sold by this agent last month. 
2. From DynamoDB, retrieve the Sales Target Data for this agent for last month.
3. Compute the penetration rate.
4. Compute the percentage of volume target achieved.
5. From DynamoDB, retrieve the commission matrix.
6. Compute the commission percentage.
7. Apply commission to all contracts.
8. Update CoreView with this new commission.

TODO: Add more details when I know how to interact with Midas and CoreView.

```python
class CommissionMatrix:
    def __init__(self, json):
        self.matrix
        self.x_axis
        self.y_axis

    def compute_commission(self, penrate: float, volume: float, sales_amount: float):
        # Use helper functions to find row and column.
        # Extract the commission percentage to be applied.
        # Apply commission percentage.
        # Return commission to add.

    def _find_row(self, penrate: float):
        # Iterate through the y-axis and find between which values the penetrationrate lands.
        # Get the index.
        # Reverse the index (y-axis index starts from the bottom left corner but that is not how matrices are indexed).
        # reverse_index = len(y_axis) - 1 - index

    def _find_column(self, volume: float):
        # Iterate through the x-axis to find where between which value the penetration index lands.
        # Get the index.
```

### Clawback lambda
The clawback lambda will be triggered by a Cronjob event the first day of every month. 

It will do the following:
1. Retrieve the clawback list from Midas.
2. Check if any of the cancelled contracts in the clawback list was cancelled within a 6 month period.
3. For all the contracts that were cancelled within a 6 month period, compute the added commission.
4. In CoreView, deduct the commission that agent earned from that contract.

### Parsing lambda
The parsing lambda will be triggered via an S3 Event Notification which occurs when a file is uploaded to the S3 bucket. The lambda will identify if the uploaded file is a Commission Matrix file or a Target Sales Volume file. This will be done by checking the name of the file. Depending on if the file is a Commission Matrix file or a Target Sales Volume file, different flows will be executed.

#### Commission Matrix file
The matrix data will be loaded with Pandas.

For every Matrix in the data, the following instructions will be executed:
1. Extract the x-axis of the Matrix.
2. Extract the y-axis of the Matrix.
3. Extract the matrix itself.
4. Construct a DynamoDB item according to the database schema.
5. Write the item to DynamoDB.

#### Target Sales Volume file
The Target Sales Volume data will be loaded with Pandas.

For every record/row in the data the following instructions will be executed:
1. Construct an item for every monthly target of the agent in that record.
2. Write those items to DynamoDB using the BatchWriteItem method.

#### Considerations
Pandas is a big library. Lambda has some size limitations; 50MB zipped and 250MB unzipped. A lightweight version of Pandas might be needed; this can be acheived with lambda layers. If the standard Pandas package fits into the size requirements, it might still be a good idea to use lambda layers to increase deployment speed.

## Database design
Volume targets will be parsed into the VolumeTargets table. Commission matrixes will be parsed into the CommissionMatrixes table. Since the partition key does not uniquely identify an item, a composite key will be used. 

![Initial draft of architecture](dbdesign.svg)

The actual matrix along with the x-axis and y-axis will be parsed into json. The x-axis refers to the penetration rate and y-axis refers to the sales volume target percentage. Below is an example of a parsed matrix:

```json
{
  "matrix": [
    [
      1.02,
      1.03,
      1.04,
      1.05
    ],
    [
      1.02,
      1.03,
      1.05,
      1.06
    ],
    [
      1.03,
      1.02,
      1.03,
      1.04
    ]
  ],
  "x-axis": [
    0.6,
    0.7,
    0.8,
    0.9
  ],
  "y-axis": [
    0.6,
    0.7,
    0.8,
    0.9
  ]
}
```

This would design would allow for easy parsing from DynamoDB into a Python Object representing the commission matrix and associated functionality:
```python
commission_matrix = CommissionMatrix.from_json(json)
commission = commission_matrix.compute_commission(penrate, achieved_volume_percentage)
```