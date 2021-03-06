TFM: Deteccción de fallos en muebles frigoríficos


## 2. Preprocesamiento de datos (Data Cleaning) <a name="Preprocesamientodeldatos"></a>

Para el procesamiento se realizaron las siguentes transformaciones 

1. [Filtrado de alarmas críticas de la tabla de alarma.](#filtradoalarma)
2. [Eliminamos outliers de la telemetría en la tabla de muebles frigoríficos y central de frío.](#outliers2)
3. [Realizamos un resample para generar las muestra y crear una data con un timestamp periorico cada 1 min.](#resampledata)
4. [Rellenamos los valores faltantes de la tabla:](#missingvalues)
    * Para Variables categóricos realizamos un .fillna(method='ffill') donde propagamos los valores con el mismo valor
    * Para Variables continuas, realizamos una interpolación de valores
5. [Realizamos un merge de las variables de cada mural con sus valores de la central de frío](#mergedata)
6. [Extraemos por medio de un merge, ventanas de 10 días de telemetría antes de las alarmas críticas filtradas en el primer paso](#mergealarmas)
7. [Generamos 2 columnas nuevas:](#lables2)
    * (Clycle_number) Una Columna para identificar los ciclos (1 para cada alarma crítica)
    * (RUL)Una para identificar por min, la cantidad de minutos restantes hasta el momento del fallo
8. [Cargamos el data set final en Google Cloud Storage](#GCS_Uploda)

![gif](https://github.com/luisnose/TFM_DeteccionFallo_CentralesFrigorifica/blob/main/Images/Limpieza_Trim.gif)



Procedemos a generar un dataset el cual será utilizado para el entrenamiento del algoritmo, de la siguiente forma:


## 1. Utilizamos la base de datos de las alarmas y filtramos de la siguiente manera:<a name="filtradoalarma"></a>

<img src=".././Images/df_alarm.png" width="100%"><br/>

* Filtramos las alarmas que habían sido generadas por murales de carne y murales de pescados ambos con controladores RX600.  

| Field name|DESCRIPTION|  
| ----------|-------|
| Example      |Alarma Alta Temperatura en servicio  |

* Filtramos el comienzo de la Alarma con la descripción Begin que cumpla la siguiente condición:    
    * No pueden haber estado inhabilitadas antes de su activación. sino la misma debería ser considerada como mantenimiento preventivo.

**Alarma Crítico**
|   |TimeStand|TYPE|    
| ----------|-------|-------|
| Example      |1|EnabledNotif |
| Example      |2|InhibitNotif|
|**Example**     |**3**|**EnabledNotif**|
| Example      |4|Being  |
| Example      |5|End  |

**Alarma Deshabilitada antes de su inicio**
|   |TimeStand|TYPE|    
| ----------|-------|-------|
| Example      |1|EnabledNotif |
| Example      |2|EnabledNotif|
| **Example**    |**3** |**InhibitNotif**|
| Example      |4|Being  |
| Example      |5|End  |

Dicho filtrado se realiza de la siguiente forma:


```
    for i, row  in df_Alarms.iterrows():
if (df_Alarms.loc[i,'ALTA_TEMPERATURA']) =='Begin' and  (df_Alarms.loc[i-1,'ALTA_TEMPERATURA'])!='InhibitNotif':
    df_Alarms.at[i,'new'] = 'True'
elif (df_Alarms.loc[i,'ALTA_TEMPERATURA']) =='Begin' and  (df_Alarms.loc[i-1,'ALTA_TEMPERATURA'])=='InhibitNotif':
    df_Alarms.at[i,'new'] = 'True but InhibitNotif by user'
else:
    df_Alarms.at[i,'new'] = 'False'     
``` 
* Filtramos las alarmas que se hayan activado dentro de una misma locación en un rango menor a 10 días, evitando considerar alarmas críticas que no hayan podido ser reparadas.

```
    for i, row  in df_Alarms.iterrows():
        Days_before = **10**
        if i == 0:
            df_Alarms.at[i,'new2'] = 'True'
        elif (df_Alarms.loc[i,'ELEMENT']) == (df_Alarms.loc[i-1,'ELEMENT']):
            if ((df_Alarms.loc[i,'TS'])-(df_Alarms.loc[i-1,'TS'])).days >= Days_before:
                df_Alarms.at[i,'new2'] = 'True'
            else:
                df_Alarms.at[i,'new2'] = 'Too Short' 
        else:
            df_Alarms.at[i,'new2'] = 'True'  
``` 

## 2. Eliminamos outliers de la telemetría en la tabla de muebles frigoríficos y central de frío:<a name="outliers2"></a>
```
#df_central = df_central.loc[(df_central['TAG_CONV_SONDA_ASP'] >-500) ]
#df_central = df_central.loc[(df_central['TAG_CONV_SONDA_COND'] >-500) ]
df_central = df_central.loc[(df_central['TAG_POTENCIA_COMP1'] >-500) ]
df_central = df_central.loc[(df_central['TAG_POTENCIA_COMP2'] >-500) ]
df_central = df_central.loc[(df_central['TAG_POTENCIA_COMP3'] >-500) ]
df_central = df_central.loc[(df_central['TAG_POTENCIA_COMP4'] >-500) ]
df_central = df_central.loc[(df_central['TAG_POTENCIA_COMP5'] >-500) ]
df_central = df_central.loc[(df_central['TAG_POTENCIA_COMP6'] >-500) ]
df_central = df_central.loc[(df_central['TAG_SETPOINT_ASP'] >-500)&(df_central['TAG_SETPOINT_ASP'] <20)  ]
df_central = df_central.loc[(df_central['TAG_SETPOINT_COND'] >-500) ]
df_central = df_central.loc[(df_central['TAG_SONDA_ASP'] >-500) ]
df_central = df_central.loc[(df_central['TAG_SONDA_COND'] >-500) ]
df_central = df_central.loc[(df_central['TAG_SONDA_TEMP_EXT'] >-10) ]
df_central = df_central.loc[(df_central['TAG_SONDA_TEMP_SUBENF'] >-10) ]
```

<a name="resampledata"></a>

## 3. Realizamos un resample para generar las muestra y crear una data con un timestamp periorico cada 1 min.

Para realizar el resample de data se tiene que agrupar por elementos por que cada uno comienza y termina en fechas distintas

```
df_loop =(df_loop.set_index('TS').groupby('ELEMENT').resample('1min').mean().drop(['ELEMENT'], 1).reset_index())
```
Esta línea es parte de una función que genera nuevas colunmas:
* 2 columnas nuevas de sumatoria tanto para petición de frío como para desescarche
* 4 columnas nuevas de delta de tiempo o cuánto tiempo ha transcurrido desde la última petición de frío o desescarche

```
def new_columns(data):
    cycles = data['cycle_number'].unique()
    df_merge_final = pd.DataFrame([])
    for i  in range(len(cycles)):
        df_loop =  data.loc[data['cycle_number']==cycles[i]]
        df_loop2 = (df_loop.loc[:, ['ELEMENT','TS','TAG_PETICION_FRIO','TAG_DESCARCHE']].set_index('TS').groupby('ELEMENT').resample('1min').mean().drop(['ELEMENT'], 1).reset_index())
        df_loop2.columns = ['ELEMENT', 'TS','SUM_PETICION_FRIO', 'SUM_DESCARCHE']

        df_loop3 = timedelta_Categorico(df_loop, 'TAG_PETICION_FRIO')
        df_loop4 = timedelta_Categorico(df_loop, 'TAG_DESCARCHE')
        
        df_loop =(df_loop.set_index('TS').groupby('ELEMENT').resample('1min').mean().drop(['ELEMENT'], 1).reset_index())
        df_loop['TAG_PETICION_FRIO'] = df_loop['TAG_PETICION_FRIO'].apply(lambda x: 1 if ( x >= 0.5 and x <= 1) else (0 if ( x < 0.5 and x >= 0) else x))
        df_loop['TAG_DESCARCHE'] = df_loop['TAG_DESCARCHE'].apply(lambda x: 1 if ( x >= 0.5 and x <= 1) else (0 if ( x < 0.5 and x >= 0) else x))
        df_loop_final = df_loop.merge(df_loop2, on=['ELEMENT','TS'], how='left')
        df_loop_final = df_loop_final.merge(df_loop3, on=['ELEMENT','TS'], how='left')
        df_loop_final = df_loop_final.merge(df_loop4, on=['ELEMENT','TS'], how='left')
        df_loop_final =(df_loop_final.set_index('TS').groupby('ELEMENT').resample('1min').mean().drop(['ELEMENT'], 1).reset_index())
        df_loop_final.loc[:,'cycle_number']=(i+1)


        
        df_merge_final = df_merge_final.append(df_loop_final).reset_index(drop=True)
    return df_merge_final
        

df = new_columns(data_frame)

```
## 4. Rellenamos los valores faltantes de la tabla:<a name="missingvalues"></a>

```
def Interpolate_data(data):
    cycles = data['cycle_number'].unique()
    df = pd.DataFrame([])
    for i  in range(len(cycles)):
        df_loop = data.loc[data['cycle_number'] == (i+1)]
        df_loop.loc[:,'TAG_SONDA_PB1'] = df_loop.loc[:,'TAG_SONDA_PB1'].interpolate()
        df_loop.loc[:,'TAG_SONDA_PB2'] = df_loop.loc[:,'TAG_SONDA_PB2'].interpolate()
        df_loop.loc[:,'TAG_PRESION_SATURACION'] = df_loop.loc[:,'TAG_PRESION_SATURACION'].interpolate()
        df_loop.loc[:,'TAG_TEMP_ASPIRACION'] = df_loop.loc[:,'TAG_TEMP_ASPIRACION'].interpolate()
        df_loop.loc[:,'TAG_RECALENT_VALVULA'] = df_loop.loc[:,'TAG_RECALENT_VALVULA'].interpolate()
        
        
        #filling the missing values in the binary columns 
        df_loop.loc[:,'TAG_APERT_VALVULA'] = df_loop.loc[:,'TAG_APERT_VALVULA'].fillna(method='ffill')
        df_loop.loc[:,'SUM_PETICION_FRIO'] = df_loop.loc[:,'SUM_PETICION_FRIO'].fillna(method='ffill')
        df_loop.loc[:,'SUM_DESCARCHE'] = df_loop.loc[:,'SUM_DESCARCHE'].fillna(method='ffill')
        df_loop.loc[:,'TAG_PETICION_FRIO'] = df_loop.loc[:,'TAG_PETICION_FRIO'].fillna(method='ffill')
        df_loop.loc[:,'TAG_DESCARCHE'] = df_loop.loc[:,'TAG_DESCARCHE'].fillna(method='ffill')
        df_loop.loc[:,'TD_0_TAG_PETICION_FRIO'] = df_loop.loc[:,'TD_0_TAG_PETICION_FRIO'].fillna(method='ffill')
        df_loop.loc[:,'TD_1_TAG_PETICION_FRIO'] = df_loop.loc[:,'TD_1_TAG_PETICION_FRIO'].fillna(method='ffill')
        df_loop.loc[:,'TD_0_TAG_DESCARCHE'] = df_loop.loc[:,'TD_0_TAG_DESCARCHE'].fillna(method='ffill')
        df_loop.loc[:,'TD_1_TAG_DESCARCHE'] = df_loop.loc[:,'TD_1_TAG_DESCARCHE'].fillna(method='ffill')

        df_loop = df_loop.dropna()
    
        df = df.append(df_loop).reset_index(drop=True)
    return(df)
    

df = Interpolate_data(data_frame)
```


<a name="mergedata"></a>

## 5. Realizamos un merge de las variables de cada mural con sus valores de la central de frío

```
def merge_variables_central(data):
    cycles = data['cycle_number'].unique()
    df_merge_final = pd.DataFrame([])
    for i  in range(len(cycles)):
        df_loop =  data.loc[data['cycle_number']==cycles[i]]
        locaciones = df_loop['ID_LOCATION'].unique()[0]
        df_loop_central = df_central.loc[df_central['ID_LOCATION']==locaciones]        
        df_loop = df_loop.merge(df_loop_central, on=['ID_LOCATION','TS'],  how='left')
        
        df_merge_final = df_merge_final.append(df_loop).reset_index(drop=True)
    return df_merge_final

df = merge_variables_central(dataframe)

```


<a name="mergealarmas"></a>

## 6. Extraemos por medio de un merge, ventanas de 10 días de telemetría antes de las alarmas críticas filtradas en el primer paso
```
def merging_data(data_telemetria, data_alarmas):
    cycles = data_alarmas['cycle_number'].unique()
    df_timeframe = pd.DataFrame([])
    for i  in range(len(cycles)):
        Alarm_cycle =data_alarmas.loc[data_alarmas['cycle_number']==cycles[i]]
        elemento = Alarm_cycle.iloc[0,0]
        t_max = Alarm_cycle['TS'].max() 
        t_min = t_max - relativedelta(days=10)
        df_loop = data_telemetria.loc[data_telemetria['ELEMENT']==elemento]
        df_loop = df_loop.loc[(df_loop['TS'] > t_min) & (df_loop['TS'] <= t_max)]
        if df_loop.shape[0] >1:
            df_loop.at[:,'cycle_number'] = (i+1)
            df_timeframe = df_timeframe.append(df_loop).reset_index(drop=True)

     
    return( df_timeframe[['TS']+['ELEMENT']+['TAG_SONDA_PB1']+['TAG_SONDA_PB2']+['TAG_PRESION_SATURACION']+
              ['TAG_TEMP_ASPIRACION']+['TAG_RECALENT_VALVULA']+['TAG_APERT_VALVULA']+['TAG_PETICION_FRIO']+
              ['TAG_DESCARCHE']+['cycle_number']])

df = merging_data(data_telemetria, data_alarmas )
```


## 7.Para Generar columna de ciclos únicos para cada alarma se realiza de la siguente forma:<a name="lables2"></a>

```
def label_Cycle(data):
    cycles = data['cycle_number'].unique()
    df = pd.DataFrame([])
    for i  in range(len(cycles)):
        df_loop = data.loc[data['cycle_number']==cycles[i]]
        
        df_loop.loc[:,'cycle_number']=(i+1)
     
        df = df.append(df_loop).reset_index(drop=True)
    return(df)
    
df = label_Cycle(data_frame)
```
```
def prepare_train_data(data, factor = 0):
    df = data.copy()
    fd_RUL = df.groupby('cycle_number')['TS'].max().reset_index()
    fd_RUL = pd.DataFrame(fd_RUL)
    fd_RUL.columns = ['cycle_number','max']
    df = df.merge(fd_RUL, on=['cycle_number'], how='left')
    df['RUL'] = ((df['max'] - df['TS']).dt.days*24 + (df['max'] - df['TS']).dt.components.hours).astype(int)
    df.drop(columns=['max'],inplace = True)
    
    return df[df['RUL'] >= factor]

df = prepare_train_data(data_frame)
```

## 8. Cargamos el data set final en Google Cloud Storage: <a name="GCS_Uploda"></a>

```
from google.cloud import storage
client = storage.Client()
bucket = client.get_bucket('table-cloud-lab-ctrlsenales-bucket')
    
bucket.blob('Dataset_mane.csv').upload_from_string(data_frame.to_csv(), 'text/csv')
print('the dataset in period minutes was load')
```

