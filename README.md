#PDFMetaDataExtractor

This is a Python tool for extracting metadata from PDFs.
It assumes that the PDFs have embedded OCR text, but it allows for the possibility of noisy OCR text.

##How it works
`PDFMetaDataExtractor` takes a two-phase approach to identifying metadata fields in documents.

The first phase is the generation of _candidates_, typically by means of a pattern-matching approach.
The idea here is to use some kind of primitive human intuitions to identify a few bits of text that probably contain the data of interest.

Once candidates have been identified, a machine learning approach can be taken to identify the correct ones, provided that training data is available.
The basic approach here is to score all candidates in training documents by their similarity to the real value and then train a regressor to use text features to predict match scores.
When a new document comes in, candidates are identified, scores are predicted using the trained regression model, and the highest (predicted) scoring candidate is chosen.

##Configuration
Configuration is via a YAML settings file. A sample settings file is provided in `settings.sample.yml`.

Fields to be extracted are specified in `fields`. 
Each key specifies a single metadata field.
A field is represented by a hash (dictionary) containing the following keys:

+   `module` - The name of the module in which the field type class is defined.
+   `class` - The name of the class for this field type. This class should extend `field.Field`
+   `parameters` (optional) - A hash of parameters to be passed to the class's constructor. 
+   `labels` - A list of labels that might be used to label this field.
+   `candidate_finders` - A hash of candidate finder definitions, explained in more detail below.
+   `features` - A hash of feature definitions, explained in more detail below.
    
A candidate finder is defined by a hash with the following keys:

-   `module` - The name of the module in which the candidate finder class is defined.
-   `class` - The name of the class for this candidate finder. This class should extend `candidate.CandidateFinder`
-   `parameters` (optional) - A hash of parameters to be passed to the class's constructor. 
  
A feature is defined by a hash with the following keys:

-   `module` - The name of the module in which the feature is defined.
-   `class` - The name of the class for this feature. This class should extend `feature.Feature`
-   `parameters` (optional) - A hash of parameters to be passed to the class's constructor.
     
Some basic field types are included in `py/fields.py`.
Some basic candidate finders are defined in `py/label_candidate_finder.py` and `py/box_phrase_candidate_finder.py`
Some basic features can be found in `py/features.py`.

The settings file should also specify the following keys:

-   `substitutions` - a hash whose keys are single characters and whose items are lists of strings likely to be substituted for those keys by the OCR software in use.
-   `files` - a hash of files. 
        So far, the only key used is `labels`, which should be a JSON file containing correct metadata values for training data.
-   `directories` a hash of directories. The following keys might be used `pickle`, `model_definition`, `csv`, `pdf`.
        The PDFs to be read should be in the `pdf` directory.
        To use `markup.py`, the `marked_pdf` directory should also be supplied.
        All directories supplied should already exist, as they will not be created automatically.
-   `db` - A hash of settings for a SQLAlchemy database engine.
        The following keys should be defined: `backend`, `username`, `password`, `server`, `port`, and (database) `name`.
        Optionally, `charset` may also be supplied.
        The only backend that has been tested so far is `pymysql` but others might work.
        More information on these parameters is available in [SQLAlchemy documentation](http://docs.sqlalchemy.org/en/latest/core/engines.html).
-   `extra_labels` - A list of label texts to be ignored for all fields. 

##Usage
Once the settings have been defined, the next step is to install the database schema.
To do this, run the `py/setup.py` script with the `--schema` flag. 
After the schema is installed, the PDF text and structure can be extracted to the database.
The PDFs should be saved in the `pdf` directory specified in the setting file, and the `labels` file should specify correct field values for all files.
Each key in the file should be a filename and values should be hashes of field_name/label pairs.
Running `python py/setup.py` will extract the PDF data to the database.
This may take a long time with a lot of files, but the extraction can be safely interrupted and restarted without causing any problems.

After setup, models can be defined and trained.
Models are defined in YAML files saved in the `model_definition` directory specified in the settings file.
A model should be a regressor that will predict match scores for candidates based on the field's features.
A model definition should include the following keys:

-   `field` : the name of the field to be predicted by this model
-   `module`: the module defining the regressor
-   `class`: the name of the regressor class
-   `threshold`: the threshold which a match score must exceed to be counted toward
-   `parameters`: parameters to be passed to the model's constructor
-   `parameter_grid`: the parameter grid to be passed to `sklearn.grid_search.GridSearchCV`
-   `folds`: the number of folds to use in cross-validation

After a model is defined, it can be trained by

    cd <project_directory>/py
    python train.py <model_definition_filename>
    
Upon running this script, the model will be cross-validated on the parameter grid, and the best estimator will be saved to the pickle directory.

To save time, candidate data can first be computed and exported using the `candidate_export.py` script.
That script will save relevant data to files in the CSV directory and output a token.
Passing this token to `train.py` with the `--token` flag will use the exported data instead of finding candidates and computing features again.

A model can be tested on the reserved test set using the `test.py` script.

Once a model has been selected, adding the `model_definition` key to the field in the settings file will allow the model to be used by calling the field's `predict` method.


##Requirements

- Python 2.7
- PDFMiner
- SQLAlchemy with at least one working database engine
- scikit learn, numpy, pandas
- fuzzywuzzy
- dateutil