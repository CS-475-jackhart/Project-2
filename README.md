# Introduction

This project will use parallelism, not for speeding data-computation, but for programming-convenience. You will create a month-by-month simulation in which each agent of the simulation will execute in its own thread where it just has to look at the state of the world around it and react to it.

You will also get to exercise your creativity by adding an additional "agent" to the simulation, one that impacts the state of the other agents and is impacted by them.

## Requirements
1. You are creating a month-by-month simulation of a field where rye-grass grows. The amount the rye grass that grows is affected by the temperature, amount of precipitation, and the number of rabbits around to eat it. The number of rabbits depends on the amount of rye grass available to eat.
2. The "state" of the system consists of the following global variables:
```C++
int	NowYear;		// 2023 - 2028
int	NowMonth;		// 0 - 11

float	NowPrecip;		// inches of rain per month
float	NowTemp;		// temperature this month
float	NowHeight;		// rye grass height in inches
int	NowNumRabbits;		// number of rabbits in the current population
```
3. Your basic time step will be one month. Interesting parameters that you need are:
```C++
const float RYEGRASS_GROWS_PER_MONTH =		20.0;
const float ONE_RABBITS_EATS_PER_MONTH =	 1.0;

const float AVG_PRECIP_PER_MONTH =	       12.0;	// average
const float AMP_PRECIP_PER_MONTH =		4.0;	// plus or minus
const float RANDOM_PRECIP =			2.0;	// plus or minus noise

const float AVG_TEMP =				60.0;	// average
const float AMP_TEMP =				20.0;	// plus or minus
const float RANDOM_TEMP =			10.0;	// plus or minus noise

const float MIDTEMP =				60.0;
const float MIDPRECIP =				14.0;
```
  Units of rye grass growth are inches.
  Units of temperature are degrees Fahrenheit (°F).
  Units of precipitation are inches.

4. Because you know ahead of time how many threads you will need (3 or 4), start the threads with a parallel sections directive:
```C++
omp_set_num_threads( 4 );	// same as # of sections
#pragma omp parallel sections
{
  #pragma omp section
  {
    Rabbits( );
  }

  #pragma omp section
  {
    RyeGrass( );
  }

  #pragma omp section
  {
    Watcher( );
  }

  #pragma omp section
  {
    MyAgent( );	// your own
  }
}       // implied barrier -- all functions must return in order
  // to allow any of them to get past here
```
5. Put this at the top of your program to make it a global:
```C++ 
unsigned int seed = 0;
```
6. The temperature and precipitation are a function of the particular month:
```C++
float ang = (  30.*(float)NowMonth + 15.  ) * ( M_PI / 180. );

float temp = AVG_TEMP - AMP_TEMP * cos( ang );
NowTemp = temp + Ranf( &seed, -RANDOM_TEMP, RANDOM_TEMP );

float precip = AVG_PRECIP_PER_MONTH + AMP_PRECIP_PER_MONTH * sin( ang );
NowPrecip = precip + Ranf( &seed,  -RANDOM_PRECIP, RANDOM_PRECIP );
if( NowPrecip < 0. )
  NowPrecip = 0.;
```

To keep this simple, a year consists of 12 months of 30 days each. The first day of winter is considered to be January 1. As you can see, the temperature and precipitation follow cosine and sine wave patterns with some randomness added.

7. Starting values are:
```C++
// starting date and time:
NowMonth =    0;
NowYear  = 2023;

// starting state (feel free to change this if you want):
NowNumRabbits = 1;
NowHeight =  5.;
```
8. In addition to this, you must add in some other phenomenon that directly or indirectly controls the growth of the rye grass and/or the rabbits' population. Your choice of this is up to you.
9. You are free to tweak the constants to make everything turn out "more interesting". 

## Use of Threads

As shown here, you will spawn three threads (four, when you add your own agent):

![alt text](https://web.engr.oregonstate.edu/~mjb/cs575/Projects/grain.thumb.jpg "")

The RyeGrass and Rabbits threads will each compute the next rye grass height and the next number of rabbits based on the current set of global state variables. They will compute these into local temporary, variables. They both then will reach the DoneComputing barrier.

At that point, both of those threads are done computing using the current set of global state variables. That means that it is safe to update the global state variables. Each thread should then copy the local variable into the global state. All 3 threads will then reach the DoneAssigning barrier.

At this point, the Watcher thread will print the current set of global state variables, increment the month, possibly increment the year, and then use the new month to compute the new Temperature and Precipitation. Note that the RyeGrass and Rabbits threads can't proceed because there is a chance they would re-compute the global state variables before they are done being printed. All 3 threads will then reach the DonePrinting barrier.

After spawning the threads, the main program should wait for the parallel sections to finish.

Each thread should return when the year hits 2029 (giving us 6 years, or 72 months, of simulation).

Remember that this description is for the core part of the project, before you add your own agent to the simulation. Adding your own agent will involve another thread and some additional interaction among the global state variables.

## Quantity Interactions

The Carrying Capacity of the rabbits is the number of inches of height of the rye grass. If the number of rabbits exceeds this value at the end of a month, decrease the number of rabbits by one. If the number of rabbits is less than this value at the end of a month, increase the number of rabbits by two, one because more grass attracts more rabbits, and another because they reproduce. :-)

Each month you will need to figure out how much the rye grass grows. If conditions are good, it will grow by RYEGRASS_GROWS_PER_MONTH. If conditions are not good, it won't.

You know how good the conditions are by seeing how close you are to an ideal temperature (°F) and precipitation (inches). Do this by computing a Temperature Factor and a Precipitation Factor like this:
![alt text](https://web.engr.oregonstate.edu/~mjb/cs575/Projects/graineqn.thumb.jpg "")
Note that there is a standard math function, exp( x ), to compute e-to-the-x:
```C++
float tempFactor = exp(   -Sqr(  ( NowTemp - MIDTEMP ) / 10.  )   );

float precipFactor = exp(   -Sqr(  ( NowPrecip - MIDPRECIP ) / 10.  )   );
```
I like squaring things with another function:
```C++
float
Sqr( float x )
{
        return x*x;
}
```

You then use tempFactor and precipFactor like this:
```C++
 float nextHeight = NowHeight;
 nextHeight += tempFactor * precipFactor * RYEGRASS_GROWS_PER_MONTH;
 nextHeight -= (float)NowNumRabbits * ONE_RABBITS_EATS_PER_MONTH;

Be sure to clamp nextHeight against zero, that is:
if( nextHeight < 0. ) nextHeight = 0.;

Something like this will work for the number of rabbits:


int nextNumRabbits = NowNumRabbits;
int carryingCapacity = (int)( NowHeight );
if( nextNumRabbits < carryingCapacity )
        nextNumRabbits++;
else
        if( nextNumRabbits > carryingCapacity )
                nextNumRabbits--;

if( nextNumRabbits < 0 )
        nextNumRabbits = 0;
```
## Structure of the Simulation Functions
Each simulation function will have a structure that looks like this:
```C++
while( NowYear < 2029 )
{
	// compute a temporary next-value for this quantity
	// based on the current state of the simulation:
	. . .

	// DoneComputing barrier:
	WaitBarrier( );	-- or --   #pragma omp barrier;
	. . .

	// DoneAssigning barrier:
	WaitBarrier( );	-- or --   #pragma omp barrier;
	. . .

	// DonePrinting barrier:
	WaitBarrier( );	-- or --   #pragma omp barrier;
	. . .
}
```

A Barrier Solution That Works for both Visual Studio and gcc/g++

You will probably get this error when doing this type of project in Visual Studio:

'#pragma omp barrier' improperly nested in a work-sharing construct

The OpenMP specifications says: "All threads in a team must execute the barrier region." Presumably this means that placing corresponding barriers in the different functions does not qualify, but somehow gcc/g++ are OK with it.

Here is a work-around. Instead of using the
#pragma omp barrier
line, use this:
WaitBarrier( );

You must call InitBarrier( n ) as part of your main program setup process, where n is the number of threads you will be waiting for at the barrier. Presumably this is 3 before you add your own quantity and is 4 after you add your own quantity.

I'm not particularly proud of this code, but it seems to work. To make this happen, first declare the following global variables:
```C++
omp_lock_t	Lock;
volatile int	NumInThreadTeam;
volatile int	NumAtBarrier;
volatile int	NumGone;

Here are the function prototypes:


void	InitBarrier( int );
void	WaitBarrier( );
```

In your main program:
```C++
omp_set_num_threads( 3 );	// or 4
InitBarrier( 3 );		// or 4
```

Here is the function code:
```C++
// specify how many threads will be in the barrier:
//	(also init's the Lock)

void
InitBarrier( int n )
{
        NumInThreadTeam = n;
        NumAtBarrier = 0;
	omp_init_lock( &Lock );
}


// have the calling thread wait here until all the other threads catch up:

void
WaitBarrier( )
{
        omp_set_lock( &Lock );
        {
                NumAtBarrier++;
                if( NumAtBarrier == NumInThreadTeam )
                {
                        NumGone = 0;
                        NumAtBarrier = 0;
                        // let all other threads get back to what they were doing
			// before this one unlocks, knowing that they might immediately
			// call WaitBarrier( ) again:
                        while( NumGone != NumInThreadTeam-1 );
                        omp_unset_lock( &Lock );
                        return;
                }
        }
        omp_unset_lock( &Lock );

        while( NumAtBarrier != 0 );	// this waits for the nth thread to arrive

        #pragma omp atomic
        NumGone++;			// this flags how many threads have returned
}
```

## Random Numbers

How you generate the randomness is up to you. As an example (which you are free to use), Joe Parallel wrote a couple of functions that return a random number between a user-given low value and a high value (note that the name overloading is a C++-ism, not a C-ism):

Note that this is using the reentrant version of rand, called rand_r.
rand, like you used in Project #1, actually keeps internal state (the current seed), which you now know is a dangerous thing. This version is safer.

If your system has the rand_r( ) function, do this:
```C++
#include <stdlib.h>
unsigned int seed = 0;


float x = Ranf( &seed, -1.f, 1.f );

. . .

float
Ranf( unsigned int *seedp,  float low, float high )
{
        float r = (float) rand_r( seedp );              // 0 - RAND_MAX

        return(   low  +  r * ( high - low ) / (float)RAND_MAX   );
}
```
If your system does not have the rand_r( ) function, do this:
```C++
#include <stdlib.h>

float x = Ranf( -1.f, 1.f );

. . .

float
Ranf( float low, float high )
{
        float r = (float) rand( );              // 0 - RAND_MAX

        return(   low  +  r * ( high - low ) / (float)RAND_MAX   );
}
```
## Results

Turn in your source code and your PDF writeup into Teach. Your writeup will consist of:
1. What your own-choice quantity was and how it fits into the simulation.
2. A table showing values for temperature, precipitation, number of rabbits, height of the rye grass, and your own-choice quantity as a function of month number.
3. A graph showing temperature, precipitation, number of rabbits, height of the rye grass, and your own-choice quantity as a function of month number. Note: if you change the units to °C and centimeters, the quantities might fit better on the same set of axes
    cm = inches * 2.54
    °C = (5./9.)*(°F-32)

    This will make your heights have larger numbers and your temperatures have smaller numbers.

4. A commentary about the patterns in the graph and why they turned out that way. What evidence in the curves proves that your own quantity is actually affecting the simulation correctly? 

Example Output
![alt text](https://web.engr.oregonstate.edu/~mjb/cs575/Projects/rabbits.thumb.jpg "")
