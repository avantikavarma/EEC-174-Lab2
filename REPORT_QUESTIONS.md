# Lab 2: Written report questions

Answer the following questions in your written report. For questions about model performance, refer to results from your own notebook and report the relevant metrics.

## 1. Data and labels

1. **Prediction task:** What does one row of the dataset represent? What are the three classes predicted in this lab, and how are they constructed from the BSA and DTA annotations?


Each row is one breath from a patient, with measurements of airflow and pressure. The lab predicts three classes: non-PVA (healthy breaths), BSA (Breath Stacking Asynchrony, where a single breath traps air in the chest), and DTA (Double Trigger Asynchrony, where two breaths in a row trap air). The labels come from the doctors’ annotation file, which has separate columns for BSA and DTA. A breath with neither label is non-PVA.

2. **Ambiguous labels:** The notebook removes breaths annotated as both BSA and DTA. Why are these rows a problem for the three-class target? Describe one alternative to removing them.


I removed breaths annotated as both BSA and DTA, from both the training and test sets. These rows can’t be placed cleanly into one class, so the model would learn from conflicting examples. One alternative is to keep them as a fourth class, so the model could learn to recognize that overlap. Another is to pick one label based on a clear rule, such as keeping the more serious one.


3. **Patient identifiers:** Why are `patient` and `breathid` excluded from the model inputs? Does excluding them guarantee that the model will generalize to patients it has never seen? Explain.


I excluded patient and breath_id because they identify who a breath came from or which breath it is, not what the breath looks like. If the model could use the patient ID, it might memorize one patient’s habits instead of learning what BSA and DTA look like. Excluding these columns doesn’t fully guarantee the model will work on new patients, though. Other features, like a patient’s typical airflow, can still hint at who they are. A proper test needs patients the model never saw during training.

## 2. Features and preprocessing

4. **Feature extraction:** Why does the lab extract features from flow and pressure waveforms before training a classifier? Identify two features that capture different aspects of a breath, and explain what each contributes.

A breath’s raw waveform is a long list of numbers, so I summarized each breath with a few features. Two examples are:

max_flow, the highest airflow during the breath, which shows how strong the breath is.
i_time, the inspiratory time, or how long the patient is breathing in. It can reveal breaths that are too short or too long.


5. **Feature scaling:** Which scaling method did you use? Why should it be fitted on the training data and then applied to the validation data, rather than fitted on both datasets together?


I used StandardScaler. I fit it only on the training data, so it learned the averages and spreads from training. I then applied those same numbers to the validation data. If I had fit it on both sets together, information from the validation data would leak into training, and my scores would look better than the model really is. In cross-validation, I also fit the scaler inside each fold for the same reason.

## 3. Model performance

6. **Random forest:** In your own words, how does a random forest combine decision trees to make a prediction? Give one reason it is a reasonable choice for this dataset.

A random forest builds many decision trees, each trained on a random portion of the data and features. Each tree makes a prediction, and the forest picks the answer most trees agree on. I think it’s a good choice here because it handles many features well, doesn’t need scaling, and is less likely to overfit than a single tree.


7. **Class-specific errors:** Examine your random forest's classification report. Which class has the lowest recall, and what kinds of mistakes does that indicate? Why might accuracy alone hide this problem?


DTA had the lowest recall. Averaged over 10 runs of my random forest, its recall was 0.117 (about 12%), with precision 0.574 and F1 0.193. BSA recall was 0.939. This means the model misses most DTA breaths, labeling them as non-PVA or BSA. Accuracy hid this problem. The model was about 86% accurate, but DTA is only about 11% of the test breaths (104 of 948). A model could look accurate while missing nearly all DTA breaths.

8. **Precision and recall:** For detecting BSA or DTA, what is the difference between a false positive and a false negative? Explain a situation in which you would prioritize recall over precision for one of these classes.


A false positive is when the model says a breath is BSA or DTA when it isn’t, which is a false alarm. A false negative is when it misses a real BSA or DTA breath. I would prioritize recall if missing a case could harm a patient, since a false alarm only means a clinician checks it again.

9. **Model comparison:** Compare the random forest with the other supervised-learning model you implemented. Which has the higher F1-score? Does the same model perform best for every class? Support your answer with your results.


My random forest had a macro F1 of 0.68 and a misclassification rate of 0.137. My L1 logistic regression had a macro F1 of 0.64 and a misclassification rate of 0.191. kNN was much worse, with a macro F1 of 0.49. So the random forest had the higher overall F1. The best model depended on the class, though. For BSA, the random forest was clearly better (F1 0.90 vs. 0.74). For DTA, logistic regression was slightly better (F1 0.27 vs. 0.23), though both were poor. These are single-run numbers from the notebook’s comparison table.

## 4. Model selection and validation

10. **Feature selection:** Compare performance using all available breath features with performance using selected features or the lab's expert-defined rules. Did the reduced feature set improve validation performance? Why might a feature that appears useful on the training set fail to improve validation performance?

I compared several feature sets on the test set:

Chi-squared kept only two features, tve and ep_auc. It performed poorly, with a macro F1 of 0.44 and a DTA F1 of 0.05.
Mutual information kept the top six features and got a macro F1 of 0.61.
Expert rules from the lab, which use the tve/tvi ratio, e_time, and tve, got a macro F1 of 0.96 and accuracy of 0.96.

The expert rules improved performance the most, and the reduced feature sets didn’t help. Chi-squared checks each feature on its own, so a feature can look useful in training without separating DTA once the model combines features.

11. **Cross-validation:** What does stratification preserve in 5-fold cross-validation? Report your F1-score for each fold and its average. What would substantial differences between folds suggest?


used stratified 5-fold cross-validation with L1 logistic regression, scaling inside each fold. My macro F1-scores were: Fold 1 = 0.872, Fold 2 = 0.911, Fold 3 = 0.893, Fold 4 = 0.934, Fold 5 = 0.904. The average F1 was 0.903, with average precision 0.954 and recall 0.867. The folds were fairly close, so the results are consistent. These scores are higher than the test-set results, likely because the folds mix breaths from the same patients.

12. **Patient-level validation:** If breaths from the same patient appear in both training and validation sets, how might that affect the reported performance? Describe a validation strategy for estimating performance on patients the model has not seen before.

If breaths from the same patient appear in both training and validation, the model may recognize that patient’s habits, which inflates the score. My cross-validation has this problem, which is likely why its scores were higher than my test-set results. To estimate performance on new patients, I would split by patient so every breath from one patient stays in the same fold. GroupKFold does this. The validation scores would then reflect patients the model never saw during training.
