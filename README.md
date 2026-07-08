# STATUS (2026 modernization):
* Builds on VS2022 (v143) + Vulkan SDK 1.4.x. 32-bit vulkan-1.lib is
  vendored (LunarG dropped 32-bit SDK components); validation layers
  unavailable on x86 — use RenderDoc.
* FIXED: _accum / _currentRender effects (layout transitions, resume
  pass depth/stencil loss, GL-era t-coord inversion in PlayerView)
* FIXED: screenshots implemented (swapchain readback)
* FIXED: swapchain OUT_OF_DATE/SUBOPTIMAL handling; vid_restart and
  resolution-change crashes (allocator deferred-free bugs)
* FIXED: stencil shadow flicker (dynamic state after pipeline bind;
  slope-scaled shadow bias for float depth — new cvar defaults)
* FIXED: demand-loaded textures black on first frame (staging flush
  before frame submit)
* KNOWN ISSUE: SWF elements glitch during menu tween animations
  (black/displaced triangles); settled UI is correct. Not stencil
  masks (audited), not texture latency, not memory coherence, not
  threading (all tested). Next step for whoever picks it up: RenderDoc
  capture mid-tween, inspect VS Input of a broken draw.
* Render debug (r_show*) still largely unimplemented.

# Updates to the original vkDoom 3 as of 07/07/2026

- Fix GL_CopyFrameBuffer — _accum and _currentRender effects

Two bugs, broken since initial release (2017):

1. The blit in GL_CopyFrameBuffer read the swapchain image declaring
   TRANSFER_SRC_OPTIMAL while it was actually in GENERAL (the render
   pass finalLayout). Added the missing transitions around the blit;
   src x/y offsets are now honored as well.

2. renderPassResume reused the main pass attachment descriptions,
   leaving depth at loadOp=DONT_CARE / initialLayout=UNDEFINED — every
   mid-frame framebuffer copy discarded the frame's depth and stencil
   for everything drawn afterward. Depth (and the MSAA color target
   when multisampling) now LOAD with correct initial layouts; the
   color attachment's initialLayout corrected from SHADER_READ_ONLY
   to GENERAL to match the actual image state.

- Fix bloodorb/_accum effects — PlayerView t-coordinate inversion

Third and final layer of the _accum fix. FullscreenFX AccumPass drew the
capture materials with inverted t-coordinates (t: 1->0) — the GL-era
compensation for glCopyTexImage2D's bottom-up captures. The Vulkan
backend captures top-down, so the compensation itself became the bug:
each accumulation generation landed vertically mirrored before
recapture, producing a mirrored, non-swirling overlay (the alternating
mirror shredded the rotation trail).

- Fixed t to normal orientation (0->1) in AccumPass; same fix applied to
the g_testBloom dev loop. Unconditional edit (game-d3xp doesn't define
ID_VULKAN); the GL build configuration would need this re-inverted.

Combined with the earlier layout-transition and resume-pass fixes,
berserk/bloodorb now renders correctly: upright, coherent swirl,
matching the GL renderer. Broken since initial release.

- Fix screenshot in VK

- Fix allocator crashes on vid_restart / resolution change + Swapchain invalidation handling

Three compounding issues in the custom allocator's deferred-free path,
exposed by mode changes through the menu (vid_restart):

1. DestroyRenderTargets unconditionally freed m_msaaAllocation; with
   r_multiSamples 0 that allocation is default-constructed with
   block == NULL, poisoning the garbage ring (null this in
   idVulkanBlock::Free). Now guarded at both the producer
   (DestroyRenderTargets) and the sink (idVulkanAllocator::Free).

2. EmptyGarbage deleted blocks that became empty mid-drain while
   later garbage entries — same list or other ring slots — still
   referenced them. Use-after-free heap corruption that surfaced
   downstream as vkBindImageMemory VALIDATION_FAILED during Restart.
   Empty blocks are now retained until Shutdown instead of eagerly
   deleted.

3. Restart() drained only one garbage ring slot before recreating
   render targets, leaving old-resolution allocations resident during
   reallocation. Now drains all NUM_FRAME_DATA slots for both the
   image and allocator rings after vkDeviceWaitIdle, where deferred
   freeing serves no purpose.

Verified: vid_restart at same resolution, 720p <-> 1080p <-> 1440p
fullscreen transitions, windowed <-> fullscreen cycling, mode revert,
with r_multiSamples 0 and 4.

# Updates to the original vkDoom 3 as of 07/06/2026

Modernize build for VS2022 (v143) and Vulkan SDK 1.4.x

Brings the 2017-era codebase up to current toolchain and Vulkan SDK.
Renderer targets Vulkan 1.3 API. Builds clean on VS2022 17.x with
Vulkan SDK 1.4.350.0; boots and renders via the Vulkan backend.

Build system:
- Retargeted all projects to v143 toolset / Windows 10 SDK
- Disabled warnings-as-errors for migration (v143 is far stricter
  than v140; to be revisited with a curated suppression list)
- Removed stale TypeInfo.h includes in d3xp/gamesys (leftover from
  original Doom 3; the header never existed in BFG)

Vulkan API modernization (RenderBackend_VK.cpp):
- apiVersion: VK_MAKE_VERSION(1,0,...) -> VK_API_VERSION_1_3
- Validation layer: VK_LAYER_LUNARG_standard_validation ->
  VK_LAYER_KHRONOS_validation
- VK_EXT_debug_report -> VK_EXT_debug_utils (messenger callback,
  severity filtering, pNext-chained into instance creation so
  vkCreateInstance-time messages are captured)
- Removed VK_RESULT_BEGIN_RANGE / VK_RESULT_RANGE_SIZE from
  VK_ErrorToString (enums deleted from modern headers)

32-bit Vulkan link library:
- LunarG removed 32-bit components from the Windows SDK as of
  1.4.304; this solution is Win32-only. vulkan-1.lib (x86) is now
  built from KhronosGroup/Vulkan-Loader source (cmake -A Win32)
  and vendored at neo/libs/vulkan/Lib32/. Linker repointed there.
- Note: 32-bit VK_LAYER_KHRONOS_validation no longer ships either,
  so r_vkEnableValidationLayers must stay 0 on modern systems
  (ValidateValidationLayers fatal-errors otherwise). Use RenderDoc
  for correctness work.

Shader fix:
- heatHazeWithMaskAndVertex.vert declared in_Color/in_Color2 at
  locations 2/3, aliasing in_Normal/in_Tangent. Modern glslang
  rejects this; old glslang silently accepted it, meaning the
  shipped SPIR-V read joint indices from normals and skin weights
  from tangents -- GPU skinning on this effect was broken since
  release. Corrected to locations 4/5 (matches LAYOUT_DRAW_VERT).
- All SPIR-V recompiled with SDK 1.4.350 glslangValidator.

Robustness:
- idRenderProgManager::Shutdown now null-guards m_parmBuffers so
  asset-load errors during Init produce the intended error dialog
  instead of a null-deref in the error handler's cleanup path.

Not included (deferred):
- VMA remains the bundled 2017 header; the build uses the custom
  allocator path (ID_USE_AMD_ALLOCATOR off). Migration to VMA 3.x
  is written up but only needed if that path is ever enabled.
  
# Archived
I have not had the time to maintain this to keep pace as a learning tool for Vulkan. There are still some good nuggets in here for beginners and it's a sizable chunk of code utilizing the API. However, it's years behind the API updates so it's not going to inform you on recent best practices. I recommend getting involved with https://github.com/RobertBeckebans/RBDOOM-3-BFG as it supports Vulkan and is actively maintained. 

# Overview
vkDOOM3 adds a Vulkan renderer to the GPL DOOM 3 BFG Edition.  It was written as an example of how to use Vulkan for writing something more sizable than simple recipes.  It covers topics such as General Setup, Proper Memory & Resource Allocation, Synchronization, Pipelines, etc.  Note this is an unofficial release.

NOTE: For those just wanting to dive straight into the Vulkan code you can find it here... neo/renderer/Vulkan.  Also you can just search for ID_VULKAN.

## TODO

As of initial release all maps load and are playable.  However, the code base is not in complete parity with the OpenGL renderer.  Here are some notes on what is still left to do or address.

* Anything using _accum is broken.
* Some SWF masks will render black in a few places.
* Render debug functionality is largely missing.  Compare RenderDebug_GL to RenderDebug_VK.
* Screenshots aren't implemented atm.
* The window will resize and go from windowed <-> fullscreen properly. But not all swapchain invalidation cases are handled.
* Some cvars from pure BFG may not function as expected.

# Building

## Windows

Prerequisites:

* [Git for Windows](https://github.com/git-for-windows/git/releases)
* [Vulkan SDK](https://vulkan.lunarg.com/) (several versions tested, but best to go with latest)
* A [Vulkan-capable GPU](https://en.wikipedia.org/wiki/Vulkan_(API)#Compatibility) with the appropriate drivers installed
* [DirectX SDK June 2010](https://www.microsoft.com/en-us/download/details.aspx?id=6812)

Start `Git Bash` and clone the vkDOOM3 repo:

~~~
git clone https://github.com/DustinHLand/vkDOOM3.git
~~~

### Visual Studio

Install [Visual Studio Community](https://www.visualstudio.com) with Visual C++ component. (Make sure you also check MFC/ATL in individual components. Community doesn't enabled this by default, and you may have issues compiling.)

Open the Visual Studio solution, `neo\doom3.sln`, select the desired configuration and platform, then
build the solution. (GL=OpenGL, VK=Vulkan)

Note: I have been working with VS2015 and have not tried 2017 yet. Contributions welcome. :)

### MinGW

Not currently supported.

## Linux

Not currently supported.

# Usage

## Game data and patching

This source release does not contain any game data, the game data is still
covered by the original EULA and must be obeyed as usual.

You must patch the game to the latest version.

Note that Doom 3 BFG Edition is available from the Steam store at
http://store.steampowered.com/app/208200/

## Setup

You can copy the assets over your repo (.gitignore is setup to handle this), or you can pass +set fs_basePath <path_to_steam_assets> on the command line.

## Steam

The Doom 3 BFG Edition GPL Source Code release does not include functionality for integrating with 
Steam.  This includes roaming profiles, achievements, leaderboards, matchmaking, the overlay, or
any other Steam features.


## Bink

The Doom 3 BFG Edition GPL Source Code release does not include functionality for rendering Bink Videos.


## Back End Rendering of Stencil Shadows

The Doom 3 BFG Edition GPL Source Code release does not include functionality enabling rendering
of stencil shadows via the "depth fail" method, a functionality commonly known as "Carmack's Reverse".

# License

See LICENSE.txt for the GNU GENERAL PUBLIC LICENSE

ADDITIONAL TERMS:  The Doom 3 BFG Edition GPL Source Code is also subject to certain additional terms. You should have received a copy of these additional terms immediately following the terms and conditions of the GNU GPL which accompanied the Doom 3 BFG Edition GPL Source Code.  If not, please request a copy in writing from id Software at id Software LLC, c/o ZeniMax Media Inc., Suite 120, Rockville, Maryland 20850 USA.

EXCLUDED CODE:  The code described below and contained in the Doom 3 BFG Edition GPL Source Code release is not part of the Program covered by the GPL and is expressly excluded from its terms.  You are solely responsible for obtaining from the copyright holder a license for such code and complying with the applicable license terms.

## VMA

neo/renderer/Vulkan/vma.h

Copyright (c) 2017 Advanced Micro Devices, Inc. All rights reserved.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.


## JPEG library

neo/renderer/jpeg-6/*

Copyright (C) 1991-1995, Thomas G. Lane

Permission is hereby granted to use, copy, modify, and distribute this
software (or portions thereof) for any purpose, without fee, subject to these
conditions:
(1) If any part of the source code for this software is distributed, then this
README file must be included, with this copyright and no-warranty notice
unaltered; and any additions, deletions, or changes to the original files
must be clearly indicated in accompanying documentation.
(2) If only executable code is distributed, then the accompanying
documentation must state that "this software is based in part on the work of
the Independent JPEG Group".
(3) Permission for use of this software is granted only if the user accepts
full responsibility for any undesirable consequences; the authors accept
NO LIABILITY for damages of any kind.

These conditions apply to any software derived from or based on the IJG code,
not just to the unmodified library.  If you use our work, you ought to
acknowledge us.

NOTE: unfortunately the README that came with our copy of the library has
been lost, so the one from release 6b is included instead. There are a few
'glue type' modifications to the library to make it easier to use from
the engine, but otherwise the dependency can be easily cleaned up to a
better release of the library.

## zlib library

neo/framework/zlib/*

Copyright (C) 1995-2005 Jean-loup Gailly and Mark Adler

This software is provided 'as-is', without any express or implied
warranty.  In no event will the authors be held liable for any damages
arising from the use of this software.

Permission is granted to anyone to use this software for any purpose,
including commercial applications, and to alter it and redistribute it
freely, subject to the following restrictions:

1. The origin of this software must not be misrepresented; you must not
 claim that you wrote the original software. If you use this software
 in a product, an acknowledgment in the product documentation would be
 appreciated but is not required.
2. Altered source versions must be plainly marked as such, and must not be
 misrepresented as being the original software.
3. This notice may not be removed or altered from any source distribution.

## Base64 implementation

neo/idlib/Base64.cpp

Copyright (c) 1996 Lars Wirzenius.  All rights reserved.

June 14 2003: TTimo <ttimo@idsoftware.com>
	modified + endian bug fixes
	http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=197039

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE AUTHOR ``AS IS'' AND ANY EXPRESS OR
IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED.  IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY DIRECT,
INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.

## IO for uncompress .zip files using zlib

neo/framework/Unzip.cpp
neo/framework/Unzip.h

Copyright (C) 1998 Gilles Vollant
zlib is Copyright (C) 1995-1998 Jean-loup Gailly and Mark Adler

  This software is provided 'as-is', without any express or implied
  warranty.  In no event will the authors be held liable for any damages
  arising from the use of this software.

  Permission is granted to anyone to use this software for any purpose,
  including commercial applications, and to alter it and redistribute it
  freely, subject to the following restrictions:

  1. The origin of this software must not be misrepresented; you must not
     claim that you wrote the original software. If you use this software
     in a product, an acknowledgment in the product documentation would be
     appreciated but is not required.
  2. Altered source versions must be plainly marked as such, and must not be
     misrepresented as being the original software.
  3. This notice may not be removed or altered from any source distribution.

## MD4 Message-Digest Algorithm

neo/idlib/hashing/MD4.cpp
Copyright (C) 1991-2, RSA Data Security, Inc. Created 1991. All
rights reserved.

License to copy and use this software is granted provided that it
is identified as the "RSA Data Security, Inc. MD4 Message-Digest
Algorithm" in all material mentioning or referencing this software
or this function.

License is also granted to make and use derivative works provided
that such works are identified as "derived from the RSA Data
Security, Inc. MD4 Message-Digest Algorithm" in all material
mentioning or referencing the derived work.

RSA Data Security, Inc. makes no representations concerning either
the merchantability of this software or the suitability of this
software for any particular purpose. It is provided "as is"
without express or implied warranty of any kind.

These notices must be retained in any copies of any part of this
documentation and/or software.

## MD5 Message-Digest Algorithm

neo/idlib/hashing/MD5.cpp
This code implements the MD5 message-digest algorithm.
The algorithm is due to Ron Rivest.  This code was
written by Colin Plumb in 1993, no copyright is claimed.
This code is in the public domain; do with it what you wish.

## CRC32 Checksum

neo/idlib/hashing/CRC32.cpp
Copyright (C) 1995-1998 Mark Adler

## OpenGL headers

neo/renderer/OpenGL/glext.h
neo/renderer/OpenGL/wglext.h

Copyright (c) 2007-2012 The Khronos Group Inc.

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and/or associated documentation files (the
"Materials"), to deal in the Materials without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Materials, and to
permit persons to whom the Materials are furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Materials.

THE MATERIALS ARE PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
MATERIALS OR THE USE OR OTHER DEALINGS IN THE MATERIALS.

## Timidity

doomclassic/timidity/*

Copyright (c) 1995 Tuukka Toivonen 

From http://www.cgs.fi/~tt/discontinued.html :

If you'd like to continue hacking on TiMidity, feel free. I'm
hereby extending the TiMidity license agreement: you can now 
select the most convenient license for your needs from (1) the
GNU GPL, (2) the GNU LGPL, or (3) the Perl Artistic License.  
