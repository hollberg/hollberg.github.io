<h2>Predicting NFL Draft Round Based on Player's NFL Combine Performance</h2>
 
 <p><i>All data and code supporting the subsequent essay can be found at <a href="https://github.com/hollberg/nfl_combine_draft">my github page</a> (specifically, the 'draft_round_predictor.ipynb' notebook).</i></p>

 <img src="/assets/img/stats_by_position.png" alt="Look at variance between 'Line' and 'Skill' players">
 
<h3>I. Introduction</h3>
Projecting (and selecting) players for the NFL draft has become a <a href="https://www.cbssports.com/nfl/draft/mock-draft/">year-round focus</a> for certain segments of the sport's fanbase. To gain insight into collegiate player's future performance, NFL executives, bloggers, and TV talking heads spend countless hours
<ul>
 <li>Reviewing tape</li>
 <li>Making assessments of things such as "Motor", "Heart", and "Grit"</li>
 <li>Asking players questions like "If you could be any kind of animal, what would you be?</li>
 </ul>
 
<p>Which raises the question - when attempting to project a player's placement in the NFL draft, can we derive any insight out of something more quantifiable than an associate GM's hamfisted attempt at pop psychology?</p>
 
<p>In this essay, I use quantifiable performance data from the NFL Combine to see if we can better predict a player's selection round in the NFL draft.</p>
 
 
<h3>II. Data and Data Wrangling</h3>
<p>To begin this process, I collected NFL Combine performance data from <a href="https://nflcombineresults.com/">NFLCombineResults.com</a>, and collected draft history data from ESPN (example page <a href="http://insider.espn.com/nfl/draft/history/_/year/history?year=2020&round=-1&team=-1&college=-1&x=33&y=23&position=-1&procoach=-1&highschoolstate=-1&award=-1&collegecoach=-1&highschool=-1">here</a>).</p>
 
<p>The resulting datasets look like this:</p>
<b>NFL Combine Data:</b>
<img src="/assets/img/combine_data_5_rows.png">
 
<br/>
<b>NFL Draft Data:</b><br/>
<img src="/assets/img/draft_data_5_rows.png">


<p>These datasets have the raw information we need, but we'll need to merge the datasets together to pair a given player's NFL Combine statistics with their ultimate draft selection. Maybe we can just join by player name? That would require that each player name is totally unique. Are they?</p>
 
<pre><code class="python">
 df_combine['name'].value_counts(sort='descending').head(10)<br/>
 Out[7]<br/>
Brandon Williams    5<br/>
Chris Brown         5<br/>
Brian Allen         4<br/>
Mike Williams       4<br/>
Chris Jones         4<br/>
</code></pre>

<p>Nope. So we'll need to join the datasets on both player name and college to ensure uniqueness. Are the college names identical between datasets? Let's merge the two datasets on their respective 'school' columns and see what matches up...</p>
<pre><code>schools = draft_school.merge(combine_school, on='school', how='outer',
                             suffixes=['_draft', '_combine']).sort_values(by='school')</code></pre>

<img src="/assets/img/college_names.png">
<br/><br/>
So it's all a mess. Here are a few of the steps required to clean up the data:
<ul>
 <li>Remove the state indicator from school name in the "NFL Combine" dataset <br/>
  <pre>regex_replace_parens = r'\([^)]*[a-zA-Z][^)]*\)'
df_combine['school'] = df_combine['school'].str.replace(regex_replace_parens,
                                                        '', regex=True)</pre></li>
 <li>Remove "Jr./III/IV", etc. suffixes from player names from the "NFL Draft" dataset:<br/>
  <pre>regex_suffixes_to_remove = r'Jr\.$|III$|IIII$|IV$|, Jr.$'
df_draft['name'] = df_draft['name'].str.replace(regex_suffixes_to_remove,
                                                '', regex=True)</pre></li>
 <li>The "NFL Combine" dataset has 25 different position labels (like SS/Strong Safety). Remap these granular positions, and assign each position an indicator of [O]ffense or [D]efense, and an indicator of whether the position is a [S]kill position or a [L]ineman position.</li>
 </ul>
 
 
<p>Now we've standardized the player and collegiate names in each dataset and can merge them together:</p>
<pre><code>df_merged = df_combine.merge(df_draft, how='left',
                             on=['name', 'school', 'year'])

df_merged.head()</code></pre>
<img src="/assets/img/merged_5_rows.png">

<p>Now we can clean up uneccesary columns and rows of data.</p>
<p>Since our target variable is the <code>round</code> in which a player is drafted, the <code>pk(ovr)</code> column (which indicates the pick within a given round and the overall pick number) would be a huge source of <a href="https://machinelearningmastery.com/data-leakage-machine-learning/">leakage</a> in our model. Let's drop that column, along with other high cardinality columns like the school name and player name. The 60-yard shuttle metric in the NFL Combine dataset has only 1370 values, so let's drop that too.</p>

<p>As for rows to drop - let's start by dropping all Kickers, Long Snappers, Fullbacks, and Quarterbacks. There are too few <a href="https://mgoblog.com/category/tags/khalid-hill-hammer-panda">Fullbacks</a> in recent years, and Kickers, Long Snappers and QB's are outliers in terms of how raw athleticism impacts their draft standings.</p>

<p>Let's also drop any player that doesn't have at least 7 of the 10 remaining NFL Combine metrics populated</p>
<pre><code>
metrics_cols = ['height (in)', 'weight (lbs)', 'hand size (in)', 'arm length (in)',
       '40 yard', 'bench press', 'vert leap (in)', 'broad jump (in)',
       'shuttle', '3cone']
df_merged.dropna(axis=0, thresh=7,
                 subset=metrics_cols, inplace=True)</code></pre>
                 
<p>And now let's replace all remaining "Nan" values from the combine with the median value for that metric <i>within</i> that player's position group. (for example, replace a "NaN" value for a cornerback's 40-yard dash with the average value of just the conernerback group. Here's the function to apply this:</p>
<pre><code>
def group_imputer(df, grouping_col, cols_to_impute):
    """
    Impute values in a dataframe based on averages WITHIN a given category
    :param df: Pandas DataFrame 
    :param grouping_col: String, column name to group by
    :param cols_to_impute: List, column names where NaNs will be imputed
    :return: Pandas DataFrame 
    """
    entries_in_group_col = df[grouping_col].unique()
    df_group_means = df.groupby(by=grouping_col)[cols_to_impute].median()

    # Loop over input df and replace/impute values
    for entry in entries_in_group_col:
        for column in cols_to_impute:
            fill_value = df_group_means.loc[entry, column]
            mask = df[grouping_col] == entry
            df.loc[mask, column] = df.loc[mask, column].fillna(fill_value)
    return df
</code></pre>

And a few last housekeeping steps:
<ul>
 <li>Players that weren't drafted (and thus aren't in the NFL Draft dataset) have a <code>NaN</code> value for their <code>round</code> data. Replace that with a value of 8, which will be our category for undrafted players.</li>
 <li>I was unable to generate any kind of improved model when processing all players in a given cycle. But by isolating just players of [L]inement or [S]killed, the models yielded meaningful predictive results. So for the following analysis, I will drop all players listed as [S]killed in the <code>line_or_skill</code> column.</li>
 <li>Our inital dataset was imbalanced - over 54% of the players fell in the "undrafted" (round=8) category, with rounds 1-7 roughly evenly distributed around 7% of the original dataset. To adjust for this, I undersampled the "undrafted" players in the dataset to roughly equalize the draft round categories in the data.</li>
 </ul>
 
<h3>II. Modeling Approach</h3>
 <p>Following all that data wrangling, let's revisit our modeling objectives</p>
 
 <p>Our goal is to use player performance data from the NFL combine to classify the round in which the player is selected in the NFL draft (where our imputed draft round of "8" represents an undrafted player).</p>
 
 <p>After much iteration, I was unable to find or tune models that could improve on the baseline accuracy score while using a dataset containing both "Skill" and "Line" players. However, when isolating the data between "beef" (Line players) and "speed" (Skill players), several models were able to significantly outpace the baseline. Keep in mind that all subsequent steps and analysis apply to a dataset filtered to only contain Offensive Line and Defensive Line players.
 
 <p> Because I've rebalanced the dataset and our categories are now roughly evenly distributed in the remaining data, I choose to use model accuracy as my target metric. If our model can improve on the baseline accuracy, we know we're generating predictive insight from the model.</p>
 
 
<h3>III. Modeling Process</h3>
 <p>Let's begin by splitting our dataset into train and test groups:</p>
 <pre><code>target = 'round'
X = df_for_models.drop(columns=target)
y = df_for_models[target]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=.2, random_state=42)</code></pre>
 
 <p>And now let's calculate our baseline accuracy measure (the percent of our dataset comprised of the most commonly occuring draft round):
  <pre><code>  baseline = y.value_counts(normalize=True).max()
  >>> 0.151515151515</code></pre>
 
 <p>So our goal with each classification model is to beat 15.15% accuracy!</p>
 
 <p>Here's how a series of models performed:</p>
 <table>
  <tr>
    <th>Model</th>
    <th>Key Parameters</th>
    <th>Validation Accuracy Score</th>
  </tr>
  <i>
  <tr>
   <td>Baseline</td>
   <td>Most common value (%)</td>
   <td>15.15%</td>
  </tr>
  </i>
  <tr>
    <td>DecisionTreeClassifier</td>
    <td>defaults</td>
    <td>14.7%</td>
  </tr>
  <tr>
    <td>RandomForestClassifier</td>
    <td>defaults</td>
    <td>17.4%</td>
  </tr>
  <tr>
   <td>Tuned RandomForestClassifier</td>
   <td>Warm start:True, n_estimators:400, max_depth: 8, boostrap:False,</td>
   <td>22.3%</td>
  </tr>
  <tr>
    <td>XGBoost</td>
    <td>loss='deviance', n_estimators:500, max_depth: 4, subsample:1,</td>
    <td>16.2%</td>
  </tr>
  <tr>
   <td>RidgeClassifierCV</td>
   <td>cv:5, alphas:[.1,1.0,10.0]</td>
   <td>21.6%</td>
  </tr>
  <tr>
   <td>Tuned RidgeClassifierCV</td>
   <td>cv:7, class_weight:None</td>
   <td>21.9%</td>
  </tr>
</table>
 
 <p>The best performing model, with an accuracy score of 22.3%, was a (hyperparameter-tuned) RandomForestClassifier where:</p>
  <pre><code>n_estimators=400,
  max_depth=8,
  bootstrap=False</code></pre>
 
 The confusion matrix for the "winning" model looks like this:
 <img src="/assets/img/tuned_rf_conf_matrix.png">


<h3>IV. Conclusion</h3>
 <p>Maybe there's something to be said for those unquantifiable attributes of "Motor", "Heart" and "Grit". While this analysis demonstrated that raw, quantifiable athletic performance at the NFL Combine clearly provides some additional predictive power over random guessing, predicting a player's draft round with 22% accuracy wouldn't outperform a casual fan's "eye test".</p>
