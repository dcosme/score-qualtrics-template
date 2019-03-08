# score-qualtrics
Code originally developed by [John Flournoy](https://github.com/jflournoy) to score Qualtrics surveys.

If you are using this script for the first time, you need to copy clean_this_study_template.r and process_qualtrics_options_template.r to new files with the same filenames but without '_template'. These are the files the script will look for, and so you should edit the content of these two files to be consistent with your project.

This will read survey data from qualtrics, csv file rubrics, and use them to produce scored scale data. Output is currently limited to histograms, and long and wide csv files.

To run from the command line, type `sh render_qualtrics_r.sh` and then inspect the file "process_qualtrics_data.html."

## Generating a credentials file
To pull data from Qualtrics, you need a credentials file with an API token associated with your account. To create the file, follow these steps.

1. Generate an API token for Qualtrics. Follow the steps outlined [here](https://www.qualtrics.com/support/integrations/api-integration/overview/)

2. Create `credentials.yaml.DEFAULT` in the root directory of the repo

```{bash}
cd ~/Documents/code/score-qualtrics
touch credentials.yaml.DEFAULT
```

3. Add API token information to the file
```{bash}
echo "user: dcosme#oregon" >> credentials.yaml.DEFAULT
echo "token: Ik0XNNQVZFdriLEnot..." >> credentials.yaml.DEFAUL
```