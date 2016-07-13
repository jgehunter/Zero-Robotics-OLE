```cpp

//Definition of frequent status flags, for convenience
#define FUEL_LOW 0
#define HIDE_SHADOW 1
#define UPLOAD 2
#define TAKE_PICTURES 3
#define for3 for(int i=0;i<3;i++)
#define FAR_ORBIT 2
#define CLOSE_ORBIT 1

//Declaration of relevant variables
float myState[12];
float myPos[3];
float goodPOILoc[2][3];
float POILoc[3][3];
float POIPicPosition[3];
float dotProduct;
float dotProductMax;
float earth[3];
float temp[3];

int whatToDoNext;
int POINumber[2];
int closePOI;
int farPOI;
int picturesTaken;
int picturesStart;
int picturesEnd;
int convenientPOI;
int side;
int convenientOrbit;

bool firstTime;
bool isBlue;
bool calculateDistances;
bool POIUndecided;
bool wellPositioned;
bool decidedSide;

//Initial set-up sequence
void init(){
   whatToDoNext = TAKE_PICTURES;
   
   firstTime = true;
   POIUndecided = true;
   
   earth[0] = 0.64f;
   earth[1] = 0.0f;
   earth[2] = 0.0f;
   
   decidedSide = false;
}

void loop(){
    //Checks how many pictures we have at the start of a loop
    picturesStart = game.getMemoryFilled();
    //When the POI resets we once again check which POI to go to
    if(api.getTime() % 60 == 0){
        POIUndecided = true;
        calculateDistances = true;
        picturesTaken = 0;
    } 
    //We obtain all the relevant information
    obtainInformation();
    //We decide what to do
    decideWhatToDo();
    //We do whatever is apropiate
    switch(whatToDoNext){
        case FUEL_LOW :
        takePictures();
        goToShadow();
        break;
        
        case HIDE_SHADOW :
        decidedSide = false;
        goToShadow();
        break;
        
        case UPLOAD :
        upload();
        break;
        
        case TAKE_PICTURES :
        takePictures();
        break;
    }
    //If after executing the previous code we have more pictures than at the start
    //It means we have taken a picture so we increase picturesTaken and decide which
    //POI to go to
    //Also if we had 2 pictures at the beggining and 0 after the loop it means
    //we have uploaded so we calculate relative distances again
    if(game.getMemoryFilled() > picturesStart){
        picturesTaken++;
        POIUndecided = true;
    } else if(game.getMemoryFilled() == 0 && picturesStart == 2) calculateDistances = true;
}


void decideWhatToDo(){
    //Here are all the conditions that determine what the program is going to do
    //at any given moment
    if(game.getFuelRemaining() <= 5){
        whatToDoNext = FUEL_LOW;
    } else if(game.getNextFlare() <= 25 && game.getNextFlare() > 0){
        whatToDoNext = HIDE_SHADOW;
    } else if(game.getMemoryFilled() == game.getMemorySize()){
        whatToDoNext = UPLOAD;
    } else if(api.getTime() >= 165 && game.getMemoryFilled() > 0) {
        whatToDoNext = UPLOAD;
        } else whatToDoNext = TAKE_PICTURES;
}

//Function that takes care of taking pictures
void takePictures(){
    //First chooses which POI we might be interested to go to
    determinePOI();
    //Then if necessary checks which POI are closest
    if(calculateDistances){ 
        relativeDistances();
        calculateDistances = false;
    }
    //Then if undecided determines specifically which POI to go to
    if(POIUndecided){ 
        choosePOI();
        POIUndecided = false;
    }
    //Actually goes ahed to take pictures
    goTakePictures();
}

//Gets myState, myPos and determines if we are blue or red
void obtainInformation(){
    api.getMyZRState(myState);
    memcpy(myPos,myState,3*sizeof(float));
    if(firstTime) {
        if(myPos[1] > 0) isBlue = true;
        else isBlue = false;
        firstTime = false;
    }
    
}

//Depending on wether we are blue or red it gets rid of the furthestPOI from us and
//retains the closest and middle onw
void determinePOI(){
    int reverseInequality;
    for3 game.getPOILoc(POILoc[i],i);
    if(isBlue) reverseInequality = 1;
    else reverseInequality = -1;
    if(POILoc[0][1] * reverseInequality < 0) {
            memcpy(goodPOILoc,POILoc+3,3*sizeof(float));
            memcpy(goodPOILoc+3,POILoc+6,3*sizeof(float));
            POINumber[0] = 1;
            POINumber[1] = 2;
        } else if(POILoc[1][1] * reverseInequality < 0) {
            memcpy(goodPOILoc,POILoc,3*sizeof(float));
            memcpy(goodPOILoc+3,POILoc+6,3*sizeof(float));
            POINumber[0] = 0;
            POINumber[1] = 2;
        } else if(POILoc[2][1] * reverseInequality < 0) {
            memcpy(goodPOILoc,POILoc,3*sizeof(float));
            memcpy(goodPOILoc+3,POILoc,3*sizeof(float));
            POINumber[0] = 0;
            POINumber[1] = 1;
        }
}

//From the 2 POI we are interested in calculates the closest and furthest one
void relativeDistances(){
    float distance1[3];
    float distance2[3];
    float POI1Loc[3];
    float POI2Loc[3];
    float dist1;
    float dist2;
    
    memcpy(POI1Loc,goodPOILoc,3*sizeof(float));
    memcpy(POI2Loc,goodPOILoc+3,3*sizeof(float));
    mathVecSubtract(distance1,POI1Loc,myPos,3);
    mathVecSubtract(distance2,POI2Loc,myPos,3);
    
    dist1 = mathVecMagnitude(distance1,3);
    dist2 = mathVecMagnitude(distance2,3);
    
    if(dist1 < dist2){
        closePOI = POINumber[0];
        farPOI = POINumber[1];
    } else {
        closePOI = POINumber[1];
        farPOI = POINumber[0];
    }
    
}

//Depending on the pictures we have taken so far determines which
//POI and in which orbit to go to
void choosePOI(){
    switch(picturesTaken){
        case 0:
        convenientPOI = closePOI;
        convenientOrbit = FAR_ORBIT;
        break;
        
        case 1:
        convenientPOI = farPOI;
        convenientOrbit = FAR_ORBIT;
        break;
        
        case 2:
        convenientPOI = closePOI;
        convenientOrbit = CLOSE_ORBIT;
        break;
        
        case 3:
        convenientPOI = farPOI;
        convenientOrbit = CLOSE_ORBIT;
        break;
    }
}

//Takes care of actually taking the pictures
void goTakePictures(){
    float vecBetween[3];
    
    game.getPOILoc(POIPicPosition,convenientPOI);
    mathVecSubtract(vecBetween,POIPicPosition,myPos,3);
    POIPicPosition[0] = 0.0;
    POIPicPosition[2] = sqrtf(0.04 -powf(POIPicPosition[1],2));
    face(POIPicPosition);
    dotProduct = mathVecInner(myPos,vecBetween,3);
    if(convenientOrbit == 2) {
    for3 POIPicPosition[i] *= 2.25;
    } else { 
        for3 POIPicPosition[i] *= 1.95;
    }
    arcMove(POIPicPosition);
    if(convenientOrbit == 2 ) {
        if(game.alignLine(convenientPOI) && dotProduct > -0.118){
        game.takePic(convenientPOI);
    }
    } else {
        if(game.alignLine(convenientPOI) && dotProduct > -0.079){
        game.takePic(convenientPOI);
    }
    }
}

//Faces the provided position
void face(float target[]){
    float tempTarget[3];
    float attitudeVector[3];
    memcpy(tempTarget,target,3*sizeof(float));
    mathVecSubtract(attitudeVector,tempTarget,myPos,3);
    api.setAttitudeTarget(attitudeVector);
}

//Uploads the pictures
void upload(){
    face(earth);
    float tempPos[3];
    memcpy(tempPos,myPos,3*sizeof(float));
    mathVecNormalize(tempPos,3);
    for3 tempPos[i] *= 0.6;
    arcMove(tempPos);
    game.uploadPic();
}

//Moves quickly to hide in the shadow
void goToShadow(){
    float szLU[3];
    float szLL[3];
    float szRU[3];
    float szRL[3];

    szLU[0] = 0.31f;
    szLL[0] = 0.31f;
    szRU[0] = 0.31f;
    szRL[0] = 0.31f;

    szLU[1] = -0.17f;
    szLL[1] = -0.17f;
    szRU[1] = 0.17f;
    szRL[1] = 0.17f;

    szLU[2] = -0.17f;
    szLL[2] = 0.17f;
    szRU[2] = -0.17f;
    szRL[2] = 0.17f;
    if(myPos[1] > 0.0) {
    	if(myPos[2] > 0.0){
	        arcMove(szRL);
	    }
    	else {
	        arcMove(szRU);
	    }
	    	} else if(myPos[2] > 0.0) {
    	    arcMove(szLL);

	    } else {
	        arcMove(szLU);
	  	    }
    if(game.getMemoryFilled() > 0){
        face(earth);
        game.uploadPic();
    }
}

//Moves following a circular trajectory
void arcMove(float posTarget2[3])
{
    float midpoint[3] = {(myPos[0]+posTarget2[0])/2, (myPos[1]+posTarget2[1])/2, (myPos[2]+posTarget2[2])/2};
    if (mathVecMagnitude(midpoint,3) < 0.35) {
        mathVecNormalize(midpoint,3);
     	for (int i = 0; i<3; i++) {
    	 	midpoint[i] *= 0.49;
     	}
     	setPositionTarget(midpoint,1.25);
     	//DEBUG((" | Heading to waypoint | "));
    }
    else {
        setPositionTarget(posTarget2,1.25);
    }
}

//Moves quicly in a straight line
void setPositionTarget(float target[], float mult) {
    float multTarget[3];
    //We are going to tell the satellite that our final destiny is further away than it really is so it goes faster
    for (int i = 0; i < 3; i++) multTarget[i] = myPos[i] + mult * (target[i] - myPos[i]);
    api.setPositionTarget(multTarget);
}

```
