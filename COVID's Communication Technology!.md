# STACK the Flags 2020: COVID's Communication Technology!
**Category:** Internet of Things   
**Points:** 984  
**Solves:** 10  
**Description:**
We heard a rumor that COVID was leveraging that smart city's 'light' technology for communication. Find out in detail on the technology and what is being transmitted.
[iot-challenge-1.logicdata](https://drive.google.com/uc?export=download&id=1OxDGIvqplTfN9WiAm5W9OdYnguG3onzk)


## logicdata file

We are provided with a .logicdata file for the challenge. A quick google search shows that it is data from a Saleae logic analyser. We download the software from their [website](https://support.saleae.com/logic-software) to take a look at the file. 

We see what looks like a bunch of data packets, one of which looks like this:
![](https://drive.google.com/uc?export=download&id=1dY63dCOfZAGo21OK-A_Zf5azBMWmbc-U)

## NEC IR Protocol
This data looks very similar to the NEC IR Protocol
![](https://drive.google.com/uc?export=download&id=1yEE9V4L9cwoT1FRY9ZtfdpjRL1f740NB)

We can find a plugin for the Saleae software for the NEC IR protocol [here](https://github.com/kodizhuk/Salae-Logic-NEC-Analyzer), but when trying it out, it seems like it is unable to parse the data properly, showing no output, probably because of the 38kHz carrier frequency.

## Demodulating the Data

The Saleae software allows us to export the data as a csv file, which we can use to decode the data.
![](https://drive.google.com/uc?export=download&id=1MH-Q1Rjmx6B_K90rbaDYM7lqyznoeTSf)

First, we read the csv file
```
import csv

data = [] 

with  open("raw.csv") as csvFile:
	csvReader = csv.reader(csvFile)
	lineCount =  0
	
	#loop through csv file
	for row in csvReader:
		if lineCount ==  0:
			print(f'Column names are {", ".join(row)}')
			lineCount +=  1
		else:
			#append the data to a list if it is not the first row
			data.append([float(row[0]),int(row[1])])
			lineCount +=  1
```
From the Saleae software, we see that there is a 38kHz carrier signal, which we can easily remove, filtering out pulse shorter than 20 microseconds
```
for i, dataPoint in enumerate(data):
    if i != len(data)-1:
        if data[i][1] == 0:
	        #calculate length that signal is low
            timeDiff = data[i+1][0] - data[i][0]
            #remove pulse if it is less than 20 microseconds long
            if timeDiff < 0.00002:
                data[i][1] = 1
```
The results:
![](https://drive.google.com/uc?export=download&id=12D_vdUfdr3hAV0ZANpqSrXZ-Dl4OYAvy)

Now we have the demodulated signal, we clean it up by removing consecutive `1` data points:
```
toPop = []
#generate list of data points to remove
for i, dataPoint in enumerate(data):
    if i != 0:
        if data[i][1] == 1 and data[i-1][1] == 1:
            toPop.append(i)
#remove data points
for i in toPop[-1:0:-1]:
    data.pop(i)
```

Now, we convert the data from absolute time to the length of each pulse:
```
cleanedData = []
for i, dataPoint in enumerate(data):
    if i != len(data)-1:
        dataTime = data[i+1][0] - data[i][0]
        cleanedData.append([dataTime, data[i][1]])
```
## Decoding the Data
Finally, we can start decoding the data.
![](https://drive.google.com/uc?export=download&id=1gz4N7CJ9ovt2gkxBoiZkBKWZtqLZ5jQN)

We label the data points based on the NEC ir protocol shown above, giving each data point a range of timings to account for variations in the captured data:
```
#0: zero width
#1: one width
#2: leading pulse
#3: leading space

for i, dataPoint in enumerate(cleanedData):
    dataTime = dataPoint[0]
    if dataTime > 0.0087 and dataTime < 0.0092:
        cleanedData[i][0] = 2
    elif dataTime > 0.004 and dataTime < 0.005:
        cleanedData[i][0] = 3
    elif dataTime > 0.0005 and dataTime < 0.0006:
        cleanedData[i][0] = 0
    elif dataTime > 0.0015 and dataTime < 0.0018:
        cleanedData[i][0] = 1
    else:
        cleanedData[i][0] = -1
```
The data is sent in packets, which we split it into. Then, we remove the first 17 data points, which correspond to the NEC protocol address and leading pulse:
```
dataPackets = []

#split data into packet
for i, dataPoint in enumerate(cleanedData[:-67]):
    if cleanedData[i][0] == 2 and cleanedData[i+1][0] == 3:
        dataPacket = []
        for j in range(67):
            if cleanedData[i+j][1] == 0:
                dataPacket.append(cleanedData[i+j][0])
		
		#remove first 17 data points
        dataPackets.append(dataPacket[17:])
```
Now, we have decoded the data in binary and we can easily decode it:
```
rawData = []
for i in dataPackets:
    stringVals = [str(j) for j in i]
    stringData = "".join(stringVals)
    rawData += chr(int(stringData[0:8],2))
    rawData += chr(int(stringData[8:16],2))

print("".join(rawData))
```
From this, we get the flag:
```
govtech-csg{CTf_IR_NEC_20@0!_}govtech-csg{CTf_IR_NEC_20@0!_}govtech-csg{CTf_IR_NEC_20@0!_}govtech-csg{CTf_IR_NEC_20@0!_}govtech-csg{CTf_IR_NEC_20@0!_}govtech-csg{CTf_IR_NEC_20@0!_}govtec
```
