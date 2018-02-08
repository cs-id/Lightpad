/*
<metadata description="A grid of notes with MPE compatibility and scale settings for use with melodic instruments. Press the Mode button to scroll through different size grids" target="Lightpad" tags="Default;MPE;MIDI;Melodic">
<modes>
  <mode name="Default"/>
</modes>
</metadata>
*/

#heapsize: 840

//==============================================================================
/*
   Heap layout:

   === 25 x Pad ===

   0     4 byte x 25   colours
   100   1 byte x 25   note numbers

   === 24 x Touch ===

   125   1 byte x 24   corresponding pad index (0xff if none)
   149   4 byte x 24   initial x positions (for relative pitchbend)
   245   4 byte x 24   initial y positions (for relative y axis)
   341   1 byte x 24   MIDI channel assigned

   === 24 x Pitch correction ===

   365   4 byte x 24   start time ms
   461   4 byte x 24   target x
   557   4 byte x 24   current x
   653   4 byte x 24   start x

   === 16 x Channel ===

   749   1 byte x 16   touch to track for this channel (depends on tracking mode)

   === 24 x Touch ===

   765   1 byte x 24   Touch note number
   789   1 byte x 24   Touch velocity
   813   1 byte x 24   list of active touch indicies in order of first played to last played

*/
//==============================================================================

int colSize, padWidth, colSpacing, rowSize, padHeight, rowSpacing, mode, offset, timeStart, cs64;
int dimFactor, dimDelay, dimDelta;
int scaleBitmask;
int numNotesInScale;
int channelLastAssigned;
int activePads;
int pitchbendRange;
int gradientColour1;
int gradientColour2;
int octave;
int xShift, yShift;
float pitchCorrectTime;
int tonic;  // 0 = C, 1 = C#, etc.
int transpose;
int scale;  // Major;Minor;Harmonic Minor;Pentatonic Neutral;Pentatonic Major;Pentatonic Minor;Blues;Dorian;Phrygian;Lydian;Mixolydian;Locrian;Whole Tone;Arabic (A);Arabic (B);Japanese;Ryukyu;8-Tone Spanish;Chromatic;
bool hideMode;
int position;   // Split into X and Y offsets
int clusterWidth;
int clusterHeight;
int xPos;
int yPos;

bool glideLockEnabled;
int glideLockInitialNote;
float glideLockTarget;
int glideLockChannel;

//==============================================================================
void Pad_setColour (int padIndex, int colour)           { setHeapInt (padIndex * 4, colour); }
int  Pad_getColour (int padIndex)                       { return getHeapInt (padIndex * 4); }
void Pad_setNote (int padIndex, int note)               { setHeapByte (padIndex + 100, note); }
int  Pad_getNote (int padIndex)                         { return getHeapByte (padIndex + 100); }
void Pad_setActive (int padIndex, bool setActive)       { activePads = setActive ? (activePads | (1 << padIndex)) : (activePads & ~(1 << padIndex)); }
bool Pad_isActive (int padIndex)                        { return activePads & (1 << padIndex); }
bool isAnyPadActive()                                   { return activePads; }

//==============================================================================
void Touch_setPad (int touchIndex, int padIndex)        { setHeapByte (touchIndex + 125, padIndex); }
int  Touch_getPad (int touchIndex)                      { return getHeapByte (touchIndex + 125); }
void Touch_setInitialX (int touchIndex, float initialX) { setHeapInt ((touchIndex * 4) + 149, int (initialX * 1e6)); }
float Touch_getInitialX (int touchIndex)                { return float (getHeapInt ((touchIndex * 4) + 149)) / 1e6; }
void Touch_setInitialY (int touchIndex, float initialY) { setHeapInt ((touchIndex * 4) + 245, int (initialY * 1e6)); }
float Touch_getInitialY (int touchIndex)                { return float (getHeapInt ((touchIndex * 4) + 245)) / 1e6; }
void Touch_setChannel (int touchIndex, int channel)     { setHeapByte (touchIndex + 341, channel); }
int  Touch_getChannel (int touchIndex)                  { return getHeapByte (touchIndex + 341); }
void Touch_setNote (int touchIndex, int noteNumber)     { setHeapByte (touchIndex + 765, noteNumber); }
int  Touch_getNote (int touchIndex)                     { return getHeapByte (touchIndex + 765); }
void Touch_setVelocity (int touchIndex, int velocity)   { setHeapByte (touchIndex + 789, velocity); }
int  Touch_getVelocity (int touchIndex)                 { return getHeapByte (touchIndex + 789); }
void Touch_setTouchByHistory (int touchIndex, int order){ setHeapByte (order + 813, touchIndex); }
int  Touch_getTouchByHistory (int order)                { return getHeapByte (order + 813); }
//==============================================================================
int PitchCorrect_getStartTime (int touchIndex)          { return getHeapInt ((touchIndex * 4) + 365); }
void PitchCorrect_setStartTime (int touchIndex, int timeMs) { setHeapInt ((touchIndex * 4) + 365, timeMs); }
float PitchCorrect_getLastTargetX (int touchIndex)      { return float (getHeapInt ((touchIndex * 4) + 461)) / 1e6; }
void PitchCorrect_setLastTargetX (int touchIndex, float targetX) { setHeapInt ((touchIndex * 4) + 461, int (targetX * 1e6)); }

void draw(int img, int rgb, int x, int y, int w, int h)
{
    int p00 = 1 << w * h - 1;
    if (!p00) return;
    
    int i = 0;
    for (int py = 0; py < h; ++py)
	{
		for (int px = 0; px < w; ++px)
		{
		    if ((p00 >> i++) & img) fillPixel (rgb, x + px, y + py);
		}
	}
}

void panic()
{
    for (int ch = 0; ch < 16; ++ch)
    {
        for (int note = 0; note < 128; ++note)
        {
            sendNoteOff(ch, note, 127);
        }
    }
}

int getColour(int note)
{
    int c;
    
    note %= 12;
    
    if      (note == 0)  c = 0xffff0000;
    else if (note == 1)  c = 0xffbf3f00;
    else if (note == 2)  c = 0xff7f7f00;
    else if (note == 3)  c = 0xff3fbf00;
    else if (note == 4)  c = 0xff00ff00;
    else if (note == 5)  c = 0xff00bf3f;
    else if (note == 6)  c = 0xff007f7f;
    else if (note == 7)  c = 0xff003fbf;
    else if (note == 8)  c = 0xff0000ff;
    else if (note == 9)  c = 0xff3f00bf;
    else if (note == 10) c = 0xff7f007f;
    else if (note == 11) c = 0xffbf003f;
    
    return c;
}

float PitchCorrect_updateX (int touchIndex, float newX)
{
    int byteIdx = (touchIndex * 4) + 557;
    float lastX = float (getHeapInt (byteIdx)) / 1e6;
    setHeapInt (byteIdx, int (newX * 1e6));

    return lastX;
}

float PitchCorrect_getCurrentX (int touchIndex)
{
    return float (getHeapInt ((touchIndex * 4) + 557)) / 1e6;
}

float PitchCorrect_getStartX (int touchIndex)
{
    return float (getHeapInt ((touchIndex * 4) + 653)) / 1e6;
}

void PitchCorrect_setStartX (int touchIndex, float value)
{
    setHeapInt ((touchIndex * 4) + 653, int (value * 1e6));
}

//==============================================================================

void Channel_setTrackedTouch (int channel, int touchIndex)
{
    setHeapByte (channel + 749, touchIndex);
}

int Channel_getTrackedTouch (int channel)
{
    return getHeapByte (channel + 749);
}

//==============================================================================
bool isPartOfScale (int noteRelativeToTonic)
{
	int noteAsBitSet = 0x01 << (mod (noteRelativeToTonic, 12));
	return (noteAsBitSet & scaleBitmask) != 0;
}

int findNthNoteInScale (int n)
{
    int mask = 1;
    int count = 0;

    for (int pos = 0; pos < 12; ++pos)
    {
        if (scaleBitmask & mask)
        {
            if (count == n)
                return pos;

            count++;
        }

        mask <<= 1;
    }

    return -1;
}

//==============================================================================
int getTouchedPad (float x, float y)
{
	int col = int (x * 0.5 * float (colSize));
	int row = int (y * 0.5 * float (rowSize));

	return (colSize * row) + col;
}

//==============================================================================
int getNoteForPad (int padIndex)
{
    // convert pad index (starting top left) to index in note sequence (starting bottom left):
    int padRow = padIndex / colSize;
    int padCol = padIndex % rowSize;
    int noteIndex = ((colSize - 1) - padRow) * colSize + padCol;
    int lowestNoteIndex = (octave * 12) + tonic + 48  + transpose;
    int topologyShift, scaleShift;
    
    //if (mode == 6) { noteIndex = ((colSize - 1) - padRow) * (colSize + 1) + padCol; return noteIndex + lowestNoteIndex; }
    if (mode < 7) return lowestNoteIndex + padIndex;

    if (colSize == 5)
    {
        topologyShift = (xShift * 5) + (yShift * 25);
 
        if (! hideMode)
        {
            lowestNoteIndex += topologyShift;
        }
        else
        {
            lowestNoteIndex += roundDownDivide (topologyShift, numNotesInScale) * 12;
            scaleShift = topologyShift % numNotesInScale;
                  
            if (scaleShift < 0)
                scaleShift += numNotesInScale;
                
        }
    }
    else
    {
        // Drum grid            
        int notesPerRow = getClusterWidth() * colSize;

        return lowestNoteIndex + (xShift * colSize) + padCol + (((colSize - 1) - padRow) * notesPerRow) + (yShift * notesPerRow * colSize);
    }

    if (! hideMode)
        return noteIndex + lowestNoteIndex;

    return findNthNoteInScale ((noteIndex + scaleShift)  % numNotesInScale) + ((noteIndex + scaleShift) / numNotesInScale * 12) + lowestNoteIndex;
}

int roundDownDivide (int a, int b)
{
    if (a >= 0)
        return a / b; 
    else
        return (a-b+1)/b;
}


//==============================================================================
int getTrailColour (int padColour)
{
    //if (padColour == 0xff000000)
        //return 0xffaaaaaa;

    return 0xff006666;//return blendARGB (0xffffffff, 0xffffffff - padColour);//return blendARGB (0xFFFFFFFF, padColour);
}

int getTrailColourV (int padColour)
{
    //if (padColour == 0xff000000)
        //return 0xffaaaaaa;

    return 0xff333333;//return blendARGB (0xffffffff, 0xffffffff - padColour);//return blendARGB (0xFFFFFFFF, padColour);
}

//==============================================================================
void updateDimFactor()
{
	if (isAnyPadActive() || dimDelta)
	{
	    if (dimFactor < 180)
	        dimDelta = 0;//60;
	    else
	        dimDelta = 0;

		dimFactor += dimDelta;
		dimDelay = 0;//8;
	}
	else
	{
		if (--dimDelay <= 0)
		{
			dimFactor -= 24;

			if (dimFactor < 0)
				dimFactor = 0;
		}
	}
}

//==============================================================================
bool drawAbsolutePad ()
{
    if (mode != 1)
        return false;

    int high = 0xff366CC5;
    int mid = blendARGB (0xff366CC5, 0xffAA429A);
    int low = 0xffAA429A;
    int dimColour = (dimFactor << 24);

    high = blendARGB (high, dimColour);
    mid = blendARGB (mid, dimColour);
    low = blendARGB (low, dimColour);

    blendGradientRect (high, mid, low, mid, 0, 0, 15, 15);

    return true;
}

void drawPad (int x, int y, int colour, int bottomRightCornerDarkeningAmount)
{
    int dark = blendARGB (colour, bottomRightCornerDarkeningAmount << 24);
    int mid  = blendARGB (colour, (bottomRightCornerDarkeningAmount / 2) << 24);

    int w = padWidth - colSpacing;
    int h = padHeight - rowSpacing;
    blendGradientRect (colour, mid, dark, mid, x * padWidth, y * padHeight, w, h);
}

void drawPads()
{
    int padIndex = 0;

    if (drawAbsolutePad())
        return;

	for (int padY = 0; padY < rowSize; ++padY)
	{
		for (int padX = 0; padX < colSize; ++padX)
		{
		    int overlayColour = Pad_isActive (padIndex) && mode > 1 ? 0x66ffffff : (dimFactor << 24);

            drawPad (padX, padY, blendARGB (Pad_getColour (padIndex), overlayColour), 0xcc);

            ++padIndex;
		}
	}
}


//==============================================================================
void initialiseScale()
{
	if (scale == 0)        scaleBitmask = 0xab5;  // major
	else                   scaleBitmask = 0xfff;  // chromatic

    int n = scaleBitmask;
    n -= ((n >> 1) & 0x5555);
    n =  (((n >> 2) & 0x3333) + (n & 0x3333));
    n =  (((n >> 4) + n) & 0x0f0f);
    n += (n >> 8);
    numNotesInScale = n & 0x3f;
}

//==============================================================================
void initialisePads()
{
    padWidth = 15 / colSize;
	colSpacing = colSize > 1 ? (15 - colSize * padWidth) / (colSize - 1) : 0;
	padWidth += colSpacing;
    padHeight = 15 / rowSize;
    rowSpacing = rowSize > 1 ? (15 - rowSize * padHeight) / (rowSize - 1) : 0;
    padHeight += rowSpacing;
	dimFactor = 0;
	dimDelay = 12;
	activePads = 0;

    int numPads = rowSize * colSize;

    for (int padIndex = 0; padIndex < numPads; ++padIndex)
	{
        // note numbers:
        int note = getNoteForPad (padIndex);
        if (note < 0) note = 0;

        Pad_setNote (padIndex, note);

        // pad colours:
		int padColour = 0xffff0000;//0xffffffff;   // tonic = white

		int noteInScale = mod (note - tonic, 12);
		//int noteInScale = mod (note - (tonic + transpose), 12);

     	if (noteInScale != 0 && noteInScale != 2 && noteInScale != 4)
		{
		    // not the tonic!

	        if (! hideMode && ! isPartOfScale (noteInScale))
			{
		        padColour = 0xff000000;
			}
		    else
			{
				//int blend = 0xff * (noteInScale - 1) / 10;

				padColour = 0xff333300;//blendARGB (gradientColour1 | 0xff000000,
				                       //(gradientColour2 & 0x00ffffff) | (blend << 24));
			}
		}
		
		if (mode < 7) padColour = getColour(note);

        Pad_setColour (padIndex, padColour);
	}
}

//==============================================================================
void initialiseTouches()
{
    for (int touchIndex = 0; touchIndex < 24; ++touchIndex)
    {
        Touch_setPad (touchIndex, 0xff);
        Touch_setChannel (touchIndex, 0xff);
        Touch_setTouchByHistory (0xff, touchIndex);
    }
}

void initialiseChannels()
{
	for (int channel = 0; channel < 16; ++channel)
    {
	    Channel_setTrackedTouch (channel, 0xff);
    }
}

void initialiseConfig()
{
    clusterWidth = 1;
    clusterHeight = 1;

    setLocalConfigItemRange (4, -4, 6);
	setLocalConfigItemRange (7, 0, 2);
	setLocalConfigItemRange (20, 1, 6);
	setLocalConfigItemRange (22, 0, 18);

	setLocalConfigItemRange (64, 0, 1);
	setLocalConfigItemRange (65, 1, 5);
	setLocalConfigItemRange (66, 1, 5);
	cs64 = getLocalConfig(64);
	//colSize = getLocalConfig(65);
	//rowSize = getLocalConfig(66);
	
	mode = getLocalConfig(20);
	if (mode < 6) { colSize = rowSize = offset = mode; }
	else if (mode == 6) { colSize = 5; rowSize = 3; offset == mode;}
	octave = getLocalConfig(4);
	transpose = getLocalConfig(5);
	pitchbendRange = getLocalConfig(3);
	
	updateTopologyShift();

	gradientColour1 = 0x7199ff;
	gradientColour2 = 0x6fe6ff;
	pitchCorrectTime = 0.2;
//	tonic = 0;
//	scale = 0;
//	hideMode = false;
}

void initialiseGlideLock()
{
    glideLockEnabled = false;
    glideLockInitialNote = 0;
    glideLockTarget = 0.0;
    glideLockChannel = 0;
}

void initialise()
{
    setLocalConfigActiveState (0, true, true);
	setLocalConfigActiveState (1, true, true);
	setLocalConfigActiveState (2, true, true);
	setLocalConfigActiveState (3, true, true);
	setLocalConfigActiveState (4, true, true);
	setLocalConfigActiveState (5, true, true);
	setLocalConfigActiveState (6, true, true);
	setLocalConfigActiveState (7, true, true);
	setLocalConfigActiveState (10, true, true);
	setLocalConfigActiveState (11, true, true);
	setLocalConfigActiveState (12, true, true);
	setLocalConfigActiveState (13, true, true);
	setLocalConfigActiveState (14, true, true);
	setLocalConfigActiveState (15, true, true);
	setLocalConfigActiveState (16, true, true);
	setLocalConfigActiveState (17, true, true);
	setLocalConfigActiveState (18, true, true);
	setLocalConfigActiveState (20, true, true);
	setLocalConfigActiveState (22, true, true);
	setLocalConfigActiveState (23, true, true);
	setLocalConfigActiveState (30, true, true);
	setLocalConfigActiveState (31, true, true);
	setLocalConfigActiveState (32, true, true);
	setLocalConfigActiveState (21, true, true);
	setLocalConfigActiveState (64, true, true);
	setLocalConfigActiveState (65, true, true);
	setLocalConfigActiveState (66, true, true);
	
	initialiseConfig();

    if (cs64) initialiseCtrl();

    initialisePlay();
}

void initialisePlay()
{
	initialiseScale();
	initialisePads();
	initialiseTouches();
	initialiseChannels();
    initialiseGlideLock();

	useMPEDuplicateFilter (true);
}

void initialiseCtrl()
{
    
}


//==============================================================================
void repaint()
{ 
    checkConfigUpdates();

    if (mode > 4)
        updatePitchCorrection();

	clearDisplay();
	updateDimFactor();

	if (isConnectedToHost())
    {
        if (!cs64) drawPads();
        else drawCtrl();
    }

    // Overlay heatmap
    drawPressureMap();
    fadePressureMap();
    //logHex(0xff - 0x3f);
}

void drawCtrl()
{
    for (int index = 0; index < 13; ++index)
    {
        draw(0x5210, getColour(index), 0, index, 15, 1);
    }
}

void handleButtonDown (int index)
{
    timeStart = getMillisecondCounter();
    if (index == 0)
    {
        cs64 = cs64 ? 0 : 1;
        setLocalConfig (64, cs64);
        //sendConfigItemToCluster (64);
        if (!cs64) initialisePlay();
        else initialiseCtrl();
        
//         if (getNumBlocksInCurrentCluster() > 1)
//         {
//             if (getClusterIndex() > 0)
//             {
//                 cs66 = 1;
//                 if (cs65 == 1)
//                 {
//                     transpose++;
//                 }
//                 else
//                 {
//                 //cs64 = 1;
//                 //gridSize = 5;
//                 }
//             }
//             else
//             {
//                 cs65 = 1;
//                 if (cs66 == 1)
//                 {
//                     transpose--;
//                 }
//                 else
//                 {
//                     if (cs64 == 1)
//                     {
//                         cs64 = 0;
//                         gridSize = 1;
//                     }
//                     else if (gridSize == 5)
//                     {
//                         cs64 = 1;
//                     }
//                     else
//                     {
//                         gridSize++;
//                     }
//                 }
//             }
//         }
//         
//         setLocalConfig (5, transpose);
//         setLocalConfig (64, cs64);
//         setLocalConfig (65, cs65);
//         setLocalConfig (66, cs66);
//         setLocalConfig (20, gridSize);
//         initialiseScale();
//         initialisePads ();
//         sendConfigItemToCluster (20);
//         sendConfigItemToCluster (64);
//         sendConfigItemToCluster (65);
//         sendConfigItemToCluster (66);
//         sendConfigItemToCluster (5);
    }
}

void handleButtonUp (int index)
{
    int timeSpent = getMillisecondCounter() - timeStart;
    if (index == 0)
    {
        if (timeSpent > 250 && cs64)
        {
            cs64 = 0;
            setLocalConfig (64, cs64);
        }
    }
}

//==============================================================================
int getAbsPitch (int touchIndex, float x)
{
    float initialX = Touch_getInitialX (touchIndex);

    float deltaX = (x - 1.0) * 12.0;

    return getPitchWheelFromDeltaX (deltaX);
}

int getPitchwheelValue (int touchIndex, float x)
{
    float initialX = Touch_getInitialX (touchIndex);
    float scaler = (2.1 * float (colSize) / 5.0);
    float deltaX = transformPitchForHideMode (touchIndex, scaler * (x - initialX));

    if (colSize == 5)
	    deltaX = handlePitchCorrection (touchIndex, deltaX);

	return getPitchWheelFromDeltaX (deltaX);
}

int getPitchWheelFromDeltaX (float deltaX)
{
    // now convert pitchbend in semitones to 14-bit pitchwheel position:
	float pitchwheel = deltaX > 0.0
	        ? map (deltaX, 0.0, float (getLocalConfig(3)), 8192.0, 16383.0)
	        : map (deltaX, float (-getLocalConfig(3)), 0.0, 0.0, 8192.0);

	return clamp (0, 16383, int (pitchwheel));
}

float transformPitchForHideMode (int touchIndex, float deltaX)
{
    if (! hideMode)
        return deltaX;

    // interpolate between actual pitches of pads left and right to x

    int deltaXLeft = deltaX < 0 ? int (deltaX) - 1 : int (deltaX);
    int initialPadIndex = Touch_getPad (touchIndex);

    int padIndexLeft = deltaXLeft + initialPadIndex;
    int padIndexRight = padIndexLeft + 1;

    // rows are incrementing when going down, not up!
    // if padIndexLeft/Right is outside of the edges of the block, you need
    // to explicitly add/subtract two rows to compensate.
    if (mod (padIndexLeft, colSize) == colSize - 1)
    {
        if (deltaX < 0)
            padIndexLeft += 2 * colSize;

        else if (deltaX > 0)
            padIndexRight -= 2 * colSize;
    }

    float pitchLeft = getNoteForPad (padIndexLeft);
    float pitchRight = getNoteForPad (padIndexRight);

    float deltaPitch = deltaX - float (deltaXLeft);
    float pitch = (pitchLeft * (1 - deltaPitch)) + (pitchRight * deltaPitch);

    return pitch - float (Pad_getNote (initialPadIndex));
}

void updatePitchCorrection()
{
    bool mpeMode = getLocalConfig (2) == 0 ? false : true;

    for (int i = 0; i < 24; ++i)
    {
        int channel = Touch_getChannel (i);

        if (mpeMode && (channel == 0))
            continue;

        if (channel != 0xff)
        {
            float deltaX = handlePitchCorrection (i, PitchCorrect_getCurrentX (i));

            sendPitchBend (channel, getPitchWheelFromDeltaX (deltaX) + getGlideLockDelta());
        }
    }
}

int getPitchCorrectTarget (int touchIndex, float x)
{
    x = x > 0 ? x + 0.5 : x - 0.5;
    int targetDelta = int (x);

    if (hideMode)
    {
        int startPadScaleIndex = getNoteForPad (Touch_getPad (touchIndex)) - tonic;

        while (! isPartOfScale (targetDelta + startPadScaleIndex))
            x > 0 ? ++targetDelta : --targetDelta;
    }

    return targetDelta;
}

float handlePitchCorrection (int touchIndex, float deltaX)
{
	float lastX = PitchCorrect_updateX (touchIndex, deltaX);
	float targetX = getPitchCorrectTarget (touchIndex, deltaX);
	float lastTargetX = PitchCorrect_getLastTargetX (touchIndex);

	if (abs (targetX - lastTargetX) > 0.02)
	{
	    // Changed note band
		startPitchCorrection (touchIndex, deltaX, targetX, 0);
	}
	else
	{
	    // Movement within the same note band
	    float deltaThisFrame = deltaX - lastX;

		if ((deltaThisFrame < -0.02 && deltaX < targetX) || (deltaThisFrame > 0.02 && deltaX > targetX))
		{
			// Moving away from target pitch
			startPitchCorrection (touchIndex, deltaX, targetX, 100);
		}
		else if (((deltaX < targetX && deltaX > 0) || (deltaX > targetX && deltaX < 0)) && abs (deltaThisFrame) > 0.2)
		{
			// Moving towards target pitch, and overtook correction glide
			startPitchCorrection (touchIndex, deltaX, targetX, 0);
		}
	}

	return calculateAdjustedPitch (touchIndex, deltaX, targetX);
}

void startPitchCorrection (int touchIndex, float x, float targetX, int delayMs)
{
	PitchCorrect_setStartTime (touchIndex, getMillisecondCounter() + delayMs);
	PitchCorrect_setStartX (touchIndex, x);
	PitchCorrect_setLastTargetX (touchIndex, targetX);
}

float calculateAdjustedPitch (int touchIndex, float deltaX, float targetX)
{
    int startTimeMs = PitchCorrect_getStartTime (touchIndex);
	int elapsed = getMillisecondCounter() - startTimeMs;
	pitchCorrectTime = float (getLocalConfig (11)) / 127.0;
	int pitchCorrectTimeMs = int (map (pitchCorrectTime, 0.0, 1.0, 20.0, 200.0) / (hideMode ? 3 : 1));

	if (elapsed <= 0) // Waiting for delay
        return deltaX;

    if (elapsed >= pitchCorrectTimeMs) // Finished
        return targetX;

    float progress = float (elapsed) / float (pitchCorrectTimeMs);
    float adjustedX = map (progress, 0.0, 1.0, PitchCorrect_getStartX (touchIndex), targetX);

    return adjustedX;
}

//==============================================================================
int getYAxisValue (int touchIndex, float y)
{
    if (getLocalConfig (7) == 0)
        return clamp (0, 127, int (127 - int (y * 63.5)));

    if (getLocalConfig (7) == 1)
        return getYAxisBipolar (touchIndex, y);

	float initialY = Touch_getInitialY (touchIndex);
	float yDelta = initialY - y;

    y = 0.5 + (applyCurve (yDelta * 0.5));

	return clamp (0, 127, int (y * 127));
}

int getYAxisBipolar (int touchIndex, float y)
{
    float initialY = Touch_getInitialY (touchIndex);
    float yDelta = abs (y - initialY);

    y = applyCurve (yDelta * 0.5);

	return clamp (0, 127, int (y * 127));
}

// Faster with lower value
float applyCurve (float yDelta)
{
    float scaler = float (getLocalConfig (12)) / 127.0;

    if (scaler > 0.0)
        return yDelta / scaler;
    else
        return yDelta;
}

//==============================================================================
void addTouchToList (int touchIndex)
{
    int endOfList = 0;

    while ((endOfList < 24) && (Touch_getTouchByHistory (endOfList) != 0xff))
        ++endOfList;

    if (endOfList < 24)
        Touch_setTouchByHistory (touchIndex, endOfList);
}

void deleteFromTouchList (int indexToDelete)
{
    for (int i = indexToDelete; i < 23; ++i)
    {
        if (Touch_getTouchByHistory (i) == 0xff)
            return;
            
        Touch_setTouchByHistory (Touch_getTouchByHistory (i + 1), i);
    }
    
    Touch_setTouchByHistory (0xff, 23);
}

void removeTouchFromList (int touchIndex)
{
    for (int i = 0; i < 24; ++i)
    {
        int touch = Touch_getTouchByHistory (i);
        
        if (touch == 0xff)
            return;
            
        if (touch == touchIndex)
        {
            deleteFromTouchList (i);
            return;
        }
    }
}

int getNumTouchesInList()
{
    int indexInList = 0;

    while (Touch_getTouchByHistory (indexInList) != 0xff && indexInList < 24)
        ++indexInList;

    return indexInList;
}

//==============================================================================
void resetGlideLockToNote (int note, int channel)
{
    glideLockInitialNote = note;
    glideLockChannel = channel;
    glideLockTarget = 8192.0;
}

int getGlideLockRate()
{
    return int (map (float (getLocalConfig (18)), 1.0, 127.0, 16.0, 3000.0));
}

void setGlideLockTarget (int note)
{
    float delta = float (note - glideLockInitialNote);
    glideLockTarget = getPitchWheelFromDeltaX (delta);
    sendPitchBend (glideLockChannel, int (glideLockTarget), getGlideLockRate());
}

int getGlideLockDelta()
{
   return glideLockEnabled
          ? int (map (glideLockTarget, 0.0, 16383.0, -8191.0, 8192.0))
          : 0;
}

//==============================================================================
void touchStart (int touchIndex, float x, float y, float z, float vz)
{
    if (cs64) { touchStartCtrl(touchIndex, x, y, z, vz); return; }
    
    int padIndex = getTouchedPad (x, y);
    int note = clamp (0, 127, Pad_getNote (padIndex));
    int colour = Pad_getColour (padIndex);
    int channel = 0xff;
    int velocity = clamp (1, 127, int (vz * 127.0));
    int pressure = clamp (0, 127, int (z * 127.0));
    int glideLockValue = getLocalConfig (18);
    bool enableMidiNoteOn = true;

    addTouchToList (touchIndex);

    if (glideLockEnabled || ((glideLockValue > 0) && (mode > 1)))
    {
        if (! glideLockEnabled)
        {
            glideLockEnabled = true;
            channel = assignChannel (note);
            resetGlideLockToNote (note, channel);
        }
        else
        {
            channel = glideLockChannel;
            setGlideLockTarget (note);
            enableMidiNoteOn = false;
        }
    }

    if (channel == 0xff)
        channel = assignChannel (note);

    if (enableMidiNoteOn)
    {
        if (pitchbendRange > 0)
        {
            if (mode == 1)
                sendPitchBend (channel, getAbsPitch (touchIndex, x));
            else
                sendPitchBend (channel, 8192);
        }

        Touch_setInitialY (touchIndex, y);
        
        if (getLocalConfig (12))
            sendMIDI (0xb0 | channel, getLocalConfig (6), getYAxisValue (touchIndex, y));

        sendMIDI (0xd0 | channel, pressure);

        sendNoteOn (channel, note, velocity);
    }

    addPressurePoint (getTrailColourV (colour), x, y, vz * 127.0);

    Pad_setActive (padIndex, true);

    Touch_setPad (touchIndex, padIndex);
    Touch_setNote (touchIndex, note);
    Touch_setInitialX (touchIndex, x);
    Touch_setChannel (touchIndex, channel);
    Touch_setVelocity (touchIndex, velocity);

    PitchCorrect_setLastTargetX (touchIndex, 0.0);
    PitchCorrect_updateX (touchIndex, 0.0);

    Channel_setTrackedTouch (channel, touchIndex);
}

void touchStartCtrl (int touchIndex, float x, float y, float z, float vz)
{
    int padIndex = getTouchedPad (x, y);
    int note = clamp (0, 127, Pad_getNote (padIndex));
    int colour = Pad_getColour (padIndex);
    int channel = getControlChannel();
    int velocity = clamp (1, 127, int (vz * 127.0));
    int pressure = clamp (0, 127, int (z * 127.0));
    int glideLockValue = getLocalConfig (18);
    bool enableMidiNoteOn = true;

    addTouchToList (touchIndex);
    Touch_setInitialY (touchIndex, y);


    addPressurePoint (0xffffff, x, y, vz * 8.0);

    Pad_setActive (padIndex, true);

    Touch_setPad (touchIndex, padIndex);
    Touch_setInitialX (touchIndex, x);
    Touch_setChannel (touchIndex, channel);
    Touch_setVelocity (touchIndex, velocity);

    Channel_setTrackedTouch (channel, touchIndex);
}

void touchMove (int touchIndex, float x, float y, float z, float vz)
{
    if (cs64) { return; }
    
    int padIndex = Touch_getPad (touchIndex);

    if (padIndex == 0xff)
        return;  // touch was not started.

    int channel = Touch_getChannel (touchIndex);

    if (Channel_getTrackedTouch (channel) != touchIndex)
        return;  // these are not the touch messages you're looking for...

    int note = Touch_getNote (touchIndex);
    int pressure = clamp (0, 127, int (z * 127.0));

    sendMIDI (0xd0 | channel, pressure);

    // Piano Mode acts as a fret
    if (getLocalConfig (17))
    {
        int newPadIndex = getTouchedPad (x, y);
        int newNote = clamp (0, 127, Pad_getNote (newPadIndex));

        if (note != newNote)
        {
            if (! glideLockEnabled)
            {
                sendNoteOff (channel, note, 0);
                sendNoteOn (channel, newNote, Touch_getVelocity (touchIndex));
            }
            else
            {
                setGlideLockTarget (newNote);
            }
            Touch_setNote (touchIndex, newNote);
            Touch_setPad (touchIndex, newPadIndex);
            Pad_setActive (padIndex, false);
            Pad_setActive (newPadIndex, true);
        }
    }
    else
    {
        if (getLocalConfig (12))
            sendMIDI (0xb0 | channel, getLocalConfig (6), getYAxisValue (touchIndex, y));

        if (pitchbendRange > 0)
        {
            PitchCorrect_updateX (touchIndex, x);

            if (mode == 1)
                sendPitchBend (channel, getAbsPitch (touchIndex, x));
            else if (glideLockEnabled)
                sendPitchBend (channel, getPitchwheelValue (touchIndex, x) + getGlideLockDelta(), getGlideLockRate());
            else
                sendPitchBend (channel, getPitchwheelValue (touchIndex, x));
        }
    }

    int colour = Pad_getColour (padIndex);
    addPressurePoint (getTrailColour (colour), x, y, z * 8.0);
}

void touchEnd (int touchIndex, float x, float y, float z, float vz)
{
    if (cs64) { return; }
    
    int padIndex = Touch_getPad (touchIndex);

    if (padIndex == 0xff)
        return;  // touch was not started.

    int channel = Touch_getChannel (touchIndex);

    int note = Touch_getNote (touchIndex);
    int velocity = clamp (0, 127, int (vz * 127.0));

    if (glideLockEnabled)
    {
        int numEvents = getNumTouchesInList();
        int eventNum = numEvents - 1;

        while (Touch_getTouchByHistory (eventNum) != touchIndex)
            eventNum--;

        if (numEvents == 1)
        {
            glideLockEnabled = false;
            sendNoteOff (glideLockChannel, glideLockInitialNote, velocity);
            Channel_setTrackedTouch (glideLockChannel, 0xff);
            deassignChannel (glideLockInitialNote, glideLockChannel);
        }
        else if (eventNum == (numEvents - 1))
        {
            int previousTouch = Touch_getTouchByHistory (eventNum - 1);
            int previousNote  = Touch_getNote (previousTouch);

            setGlideLockTarget (previousNote);
            Channel_setTrackedTouch (glideLockChannel, previousTouch);
            Pad_setActive (Touch_getPad (previousTouch), true);
        }
    }
    else
    {
        sendNoteOff (channel, note, velocity);
        Channel_setTrackedTouch (channel, 0xff);
        deassignChannel (note, channel);
    }

    Pad_setActive (padIndex, false);

    Touch_setPad (touchIndex, 0xff);
    Touch_setChannel (touchIndex, 0xff);

    removeTouchFromList (touchIndex);
}

void updateTopologyShift ()
{
    int xShiftLast = xShift;
    int yShiftLast = yShift;
    xShift = 0;
    yShift = 0;
    
    if (getClusterWidth() > 1)
    {
        if (mode < 5)
        {
            xShift = getClusterXpos();
        }
        else if (isMasterInCurrentCluster())
        {
            xShift = getHorizontalDistFromMaster() / 2;
        }
        else
        {
            int octStart = ((getClusterWidth() - 1) / 2);
            xShift = (getClusterXpos() - octStart);
        }
    }
    
    if (getClusterHeight() > 1)
    {
        if (mode < 5)
        {
            yShift = getClusterYpos();
        }
        if (isMasterInCurrentCluster())
        {
            yShift = getVerticalDistFromMaster() / 2;
        }
        else
        {            
            int octStart = ((getClusterHeight() - 1) / 2);            
            yShift = (getClusterYpos() - octStart);
        }
    }

    if (clusterWidth != getClusterWidth() || xPos != getClusterXpos() || xShiftLast != xShift)
    {
        if (isMasterInCurrentCluster())
        {
            if (isMasterBlock())
                syncCluster();
        }
        else if (! getClusterXpos())
            syncCluster();

		initialiseConfig();
		initialiseScale();
        initialisePads();

        clusterWidth = getClusterWidth();
		xPos = getClusterXpos();
    }
    else if (clusterHeight != getClusterHeight() || yPos != getClusterYpos() || yShiftLast != yShift)
    {
        if (isMasterInCurrentCluster())
        {
            if (isMasterBlock())
                syncCluster();
        }
        else if (! getClusterYpos())
            syncCluster();

		initialiseScale();
        initialisePads();

        clusterHeight = getClusterHeight();
		yPos = getClusterYpos();
    }
}

void syncCluster()
{
    for (int i = 4; i <= 7; ++i)
        sendConfigItemToCluster (i);

    for (int i = 10; i <= 18; ++i)
        sendConfigItemToCluster (i);

    for (int i = 20; i <= 23; ++i)
        sendConfigItemToCluster (i);
        
    for (int i = 65; i <= 95; ++i)
        sendConfigItemToCluster (i);
}

void sendConfigItemToCluster (int itemId)
{
    if (getClusterWidth() < 2)
        return;

    int numBlocksInCluster = getNumBlocksInCurrentCluster();

    for (int i = 0; i < numBlocksInCluster; ++i)
        if (getBlockIdForBlockInCluster(i) != getBlockIDForIndex(0))
            setRemoteConfig (getBlockIdForBlockInCluster(i), itemId, getLocalConfig (itemId));
}

void checkConfigUpdates ()
{
    if (scale != getLocalConfig (22))   
    {
        scale = getLocalConfig (22);
        initialiseScale();
        initialisePads();
    }
    if (mode != getLocalConfig (20))
    {
        mode = getLocalConfig (20);
        initialiseConfig();
        initialiseScale();
        initialisePads();
    }
    if (octave != getLocalConfig (4))
    {
        octave = getLocalConfig (4);
        initialisePads();
    }
    if (hideMode != getLocalConfig (23))
    {
        hideMode = getLocalConfig (23);
        initialiseScale();
        initialisePads();
    }
    if (transpose != getLocalConfig (5))
    {
        transpose = getLocalConfig (5);
        initialisePads();
    }
//     if (cs64 != getLocalConfig (64))
//     {
//         cs64 = getLocalConfig (64);
//         initialiseScale();
//         initialisePads();
//     }
// //     if (colSize != getLocalConfig (65))//     if (colSize != getLocalConfig (65))
//     {
//         colSize = getLocalConfig (65);
//         initialiseScale();
//         initialisePads();
//     }
//     if (rowSize != getLocalConfig (66))
//     {
//         rowSize = getLocalConfig (66);
//         initialiseScale();
//         initialisePads();
//     }
// 
    updateTopologyShift();
}