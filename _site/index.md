# Introduction

This article covers the fundamentals of building and rendering a 3D grid of voxels with per voxel lighting effects. Expect this to be beginner friendly with a decent but not perfect next gen output at the end. I will cover the principles behind this "per voxel" idea, while also mentioning areas of improvement if you are interested in doing extra reserach on other features or implementations for better results. This should be easy to follow along regardless of what graphics API you choose, for my examples I will showcase OpenCL, since this is what I used for this project, but you can feel free to follow along with something else. 

# Storing the voxels
## But what is a voxel? 
We can declare voxels simply by using two float4 or float3 (let's go with the first option because it might help with memory aligment later). We don't really need a position for a voxel since we are going to use a grid, but we surely need a way to express how that voxel looks. The first float is for the base color, and the second one will hold the lighting color.
```cpp
struct Voxel{
    float4 color;//base color
    float4 extra;//used for color evaluation
}
```

Not having a position for each voxel might sound odd, but we can use this very basic formula to determine the index based on x,y,z positions on the grid when looping trough the grid.
```cpp
int x, y, z; 
int GRID_SIZE = 64;

int index = x + y * GRID_SIZE + z * GRID_SIZE * GRID_SIZE;
```

Before playing with cool light effects we need a way to store our voxels. Let's aim for a fixed 64x64x64 grid. For what we aim at the end this, offers enough space for some decent test scenes and will not impact performance that much.

The easiest way to store this is a simple C style array of voxels.
```cpp
Voxel grid[GRID_SIZE * GRID_SIZE * GRID_SIZE]l
```

# Rendering voxels
## Plotting pixels
We will make good use of something that has become really popular in the last couple of years RAY-TRACING;ray marching to be more precise. 

The basic idea is to have some code execute for each pixel so we can draw something. Ray marching comes into play so we can determine if that pixel is supposed to represent a part of one of our voxels or not.

Now let's take a step back and understand what is ray tracing, ray marching and how we can use this here to get nice performance and results. Both ray marching and ray tracing are ways to determine an intersection between a ray and something we define(usually by some mathematical formuals). The difference between the two is how the ray is being shot in the scene. Ray tracing is straight to the point, yields nicer looking results but is slower. Ray marching will gradually step into the scene step by step until it hits something. 
This being said, we want to have rays for each pixel on the screen, marching forward to see what we hit so we color each pixel accordingly.

## DDA Algorithm
We are going to use DDA, also know as the Fast Grid traversal algorithm for our ray marching technique. This is a really fast and precise way to march inside a grid.

How the basic principle of DDA works?
We evaluate how much the ray travles on each axis inside the voxel and we determine which voxel comes next based on the shortest axis traversed and the direction of it.
<a href="https://www.youtube.com/watch?v=NbSee-XM7WA&t=412s">This tutorial for DDA with a great visual example</a>

The main core of the algorithm in code:
```cpp
if (tMax.x < tMax.y && tMax.x < tMax.z) 
{
    currentCell.x += step.x;
    tMax.x += tDelta.x;
    //maybe pic a normal too?
    //normal = (float3)(-step.x, 0.0f, 0.0f);
} else if (tMax.y < tMax.z) {
    currentCell.y += step.y;
    tMax.y += tDelta.y;
} else {
    currentCell.z += step.z;
    tMax.z += tDelta.z;
}

//What if we hit something?
//we check if the grid holds something in there or not, if there is a solid voxel and not air, we can say that we hit something
if (currentCell.x >= 0 && currentCell.x < GRID_SIZE &&
        currentCell.y >= 0 && currentCell.y < GRID_SIZE &&
        currentCell.z >= 0 && currentCell.z < GRID_SIZE) {
        
        int indexGrid = currentCell.z * GRID_SIZE * GRID_SIZE +
                        currentCell.y * GRID_SIZE + currentCell.x;
        if (grid[indexGrid].color.w == 1.0f) {
            *normalOut = normal;
            return indexGrid; // Hit a non-empty voxel
        }
}

```
Now that we can traverse the grid, we can easily determine if a point is inside a voxel or not, giving me the ability to color that pixel whatever color I want.
```cpp
int hitIndex = march_ray(ray, grid, &normal);
if (hitIndex != -1) {
    
    //this is voxel, let's make it white!
    pixelBuffer[index + 0] = 255.0f; // Red
    pixelBuffer[index + 1] = 255.0f; // Green
    pixelBuffer[index + 2] = 255.0f; // Blue

} else {
    pixelBuffer[index + 0] = 0.0f; // Red
    pixelBuffer[index + 1] = 0.0f; // Green
    pixelBuffer[index + 2] = 0.0f; // Blue
}
```