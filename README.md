# score-qualtrics
This is a template workflow for the `scorequaltrics` R package developed by [John Flournoy](https://github.com/jflournoy/qualtrics) to score Qualtrics surveys.


## Generating a credentials file
To pull data from Qualtrics, you need a credentials file with an API token associated with your account. To create the file, follow these steps.

1. Generate an API token for Qualtrics. Follow the steps outlined [here](https://www.qualtrics.com/support/integrations/api-integration/overview/)

2. Create `credentials.yaml.DEFAULT` in the `credentialDir` and add API token information

```{bash}
credentialDir='/Users/danicosme/' #replace with your path

if [ ! -f ${credentialDir}credentials.yaml.DEFAULT ]; then
  cd ${credentialDir}
  touch credentials.yaml.DEFAULT
  echo "user: dcosme#oregon" >> credentials.yaml.DEFAULT #replace with your token information
  echo "token: IhaSx923jsjDjaSKDjh..." >> credentials.yaml.DEFAULT #replace with your token information
else
  echo "credential file already exists in this location"
fi
```
