# Music recommendation system
Frank FAN, Yue
20640052

This is my final report. I will use several models to predict the chances of a user listening to a song repetitively afterthe first observable listening event within a time window was triggered.

KKBOX provides a training data set consists of information of the first observable listening eventfor each unique user-song pair within a specific time duration. Metadata of each unique user andsong pair is also provided. 

# Data acquisition


```python
import numpy as np
import pandas as pd
#import lightgbm as lgb

print('Loading data...')
data_path = '/Users/fanyue/Desktop/data/'
train = pd.read_csv(data_path + 'train.csv', dtype={'msno' : 'category',
                                                'source_system_tab' : 'category',
                                                  'source_screen_name' : 'category',
                                                  'source_type' : 'category',
                                                  'target' : np.uint8,
                                                  'song_id' : 'category'})
test = pd.read_csv(data_path + 'test.csv', dtype={'msno' : 'category',
                                                'source_system_tab' : 'category',
                                                'source_screen_name' : 'category',
                                                'source_type' : 'category',
                                                'song_id' : 'category'})
songs = pd.read_csv(data_path + 'songs.csv',dtype={'genre_ids': 'category',
                                                  'language' : 'category',
                                                  'artist_name' : 'category',
                                                  'composer' : 'category',
                                                  'lyricist' : 'category',
                                                  'song_id' : 'category'})
members = pd.read_csv(data_path + 'members.csv',dtype={'city' : 'category',
                                                      'bd' : np.uint8,
                                                      'gender' : 'category',
                                                      'registered_via' : 'category'})
songs_extra = pd.read_csv(data_path + 'song_extra_info.csv')
```

    Loading data...


# Data preprocessing

Firstly, we have to join all the useful factors


```python
print('Data preprocessing...')
song_cols = ['song_id', 'artist_name', 'genre_ids', 'song_length', 'language']
train = train.merge(songs[song_cols], on='song_id', how='left')
test = test.merge(songs[song_cols], on='song_id', how='left')

members['registration_year'] = members['registration_init_time'].apply(lambda x: int(str(x)[0:4]))
members['registration_month'] = members['registration_init_time'].apply(lambda x: int(str(x)[4:6]))
members['registration_date'] = members['registration_init_time'].apply(lambda x: int(str(x)[6:8]))

members['expiration_year'] = members['expiration_date'].apply(lambda x: int(str(x)[0:4]))
members['expiration_month'] = members['expiration_date'].apply(lambda x: int(str(x)[4:6]))
members['expiration_date'] = members['expiration_date'].apply(lambda x: int(str(x)[6:8]))
members = members.drop(['registration_init_time'], axis=1)

def isrc_to_year(isrc):
    if type(isrc) == str:
        if int(isrc[5:7]) > 17:
            return 1900 + int(isrc[5:7])
        else:
            return 2000 + int(isrc[5:7])
    else:
        return np.nan
        
songs_extra['song_year'] = songs_extra['isrc'].apply(isrc_to_year)
songs_extra.drop(['isrc', 'name'], axis = 1, inplace = True)

train = train.merge(members, on='msno', how='left')
test = test.merge(members, on='msno', how='left')

train = train.merge(songs_extra, on = 'song_id', how = 'left')
test = test.merge(songs_extra, on = 'song_id', how = 'left')

import gc
del members, songs; gc.collect();

for col in train.columns:
    if train[col].dtype == object:
        train[col] = train[col].astype('category')
        test[col] = test[col].astype('category')

```

    Data preprocessing...


Then, I will divide the factors and object


```python
X = train.drop(['target'], axis=1)
y = train['target'].values

X_test = test.drop(['id'], axis=1)
ids = test['id'].values

del train, test; gc.collect();
```

# Model

lightgbm model


```python
import lightgbm as lgb
d_train = lgb.Dataset(X, y)
watchlist = [d_train]

#Those parameters are almost out of hat, so feel free to play with them. I can tell
#you, that if you do it right, you will get better results for sure ;)
print('Training LGBM model...')
params = {}
params['learning_rate'] = 0.2
params['application'] = 'binary'
params['max_depth'] = 8
params['num_leaves'] = 2**8
params['verbosity'] = 0
params['metric'] = 'auc'

model = lgb.train(params, train_set=d_train, num_boost_round=50, valid_sets=watchlist, \
verbose_eval=5)

print('Making predictions and saving them...')
p_test = model.predict(X_test)

subm = pd.DataFrame()
subm['id'] = ids
subm['target'] = p_test
subm.to_csv('/Users/fanyue/Desktop/data/submission.csv.gz', compression = 'gzip', index=False, float_format = '%.5f')
print('Done!')
```

    Training LGBM model...
    [5]	training's auc: 0.706984
    [10]	training's auc: 0.719556
    [15]	training's auc: 0.728737
    [20]	training's auc: 0.734537
    [25]	training's auc: 0.738759
    [30]	training's auc: 0.74146
    [35]	training's auc: 0.74411
    [40]	training's auc: 0.746589
    [45]	training's auc: 0.748792
    [50]	training's auc: 0.751141
    Making predictions and saving them...
    Done!


training auc is 0.751141 and testing auc is 0.6685
