# geometrize-ios-framework
hexa recreate
yes
name: Build Geometrize iOS Framework

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: macos-latest

    steps:
    - name: Checkout Geometrize
      uses: actions/checkout@v3
      with:
        repository: Tw1ddle/geometrize-haxe
        path: geometrize-haxe

    - name: Setup Haxe
      uses: jommyy/setup-haxe@master
      with:
        haxe-version: latest

    - name: Install Haxe Libraries
      run: |
        haxelib install hxcpp
        haxelib list

    - name: Compile Haxe to C++
      run: |
        cd geometrize-haxe
        mkdir -p build/cpp
        haxe -cp src -main Geometrize -cpp build/cpp -D real_position -dce full

    - name: Create Framework Project Structure
      run: |
        mkdir -p GeometrizeFramework/GeometrizeFramework

        # Copy Haxe-generated C++ files
        cp -r geometrize-haxe/build/cpp/src/* GeometrizeFramework/GeometrizeFramework/ || true

        # Create project directories
        mkdir -p GeometrizeFramework/GeometrizeFramework/Public
        mkdir -p GeometrizeFramework/GeometrizeFramework/Private

    - name: Create Wrapper Headers
      run: |
        cat > GeometrizeFramework/GeometrizeFramework/Public/GeometrizeWrapper.h << 'EOF'
        #ifndef GeometrizeWrapper_h
        #define GeometrizeWrapper_h

        #import <Foundation/Foundation.h>
        #import <UIKit/UIKit.h>

        NS_ASSUME_NONNULL_BEGIN

        @interface GeometrizeShape : NSObject
        @property (nonatomic, assign) CGFloat x;
        @property (nonatomic, assign) CGFloat y;
        @property (nonatomic, assign) CGFloat width;
        @property (nonatomic, assign) CGFloat height;
        @property (nonatomic, assign) CGFloat rotation;
        @property (nonatomic, strong) UIColor *color;
        @property (nonatomic, assign) NSInteger shapeType; // 0=circle, 1=rect, 2=triangle, etc
        @end

        @interface GeometrizeWrapper : NSObject

        + (NSArray<GeometrizeShape *> *)geometrizeImage:(UIImage *)image
                                             shapeCount:(NSInteger)shapeCount
                                           maxIterations:(NSInteger)iterations;

        @end

        NS_ASSUME_NONNULL_END
        EOF

    - name: Create Wrapper Implementation
      run: |
        cat > GeometrizeFramework/GeometrizeFramework/Private/GeometrizeWrapper.mm << 'EOF'
        #import "GeometrizeWrapper.h"
        #import "Geometrize.h"

        @implementation GeometrizeShape
        @end

        @implementation GeometrizeWrapper

        + (NSArray<GeometrizeShape *> *)geometrizeImage:(UIImage *)image
                                             shapeCount:(NSInteger)shapeCount
                                           maxIterations:(NSInteger)iterations {
            NSMutableArray<GeometrizeShape *> *shapes = [NSMutableArray array];

            // TODO: Implement C++ to Objective-C bridge
            // This is where you'd call the Haxe-generated C++ functions
            // and convert results to GeometrizeShape objects

            return shapes;
        }

        @end
        EOF

    - name: Create Module Map
      run: |
        mkdir -p GeometrizeFramework/GeometrizeFramework/Modules
        cat > GeometrizeFramework/GeometrizeFramework/Modules/module.modulemap << 'EOF'
        framework module GeometrizeFramework {
          umbrella header "GeometrizeFramework.h"
          export *
          module * { export * }
        }
        EOF

    - name: Create Framework Header
      run: |
        cat > GeometrizeFramework/GeometrizeFramework/Public/GeometrizeFramework.h << 'EOF'
        //
        //  GeometrizeFramework.h
        //

        #ifndef GeometrizeFramework_h
        #define GeometrizeFramework_h

        #import <UIKit/UIKit.h>

        //! Project version number for GeometrizeFramework.
        FOUNDATION_EXPORT double GeometrizeFrameworkVersionNumber;

        //! Project version string for GeometrizeFramework.
        FOUNDATION_EXPORT const unsigned char GeometrizeFrameworkVersionString[];

        #import <GeometrizeFramework/GeometrizeWrapper.h>

        #endif /* GeometrizeFramework_h */
        EOF

    - name: Create pbxproj
      run: |
        cd GeometrizeFramework
        cat > project.pbxproj.template << 'EOF'
        {
          "archiveVersion": 1,
          "classes": {},
          "objectVersion": 54,
          "objects": {},
          "rootObject": "ROOT"
        }
        EOF

    - name: Build Framework for Device (arm64)
      run: |
        cd GeometrizeFramework
        xcodebuild -scheme GeometrizeFramework \
          -configuration Release \
          -arch arm64 \
          -sdk iphoneos \
          -derivedDataPath build_device \
          BUILD_LIBRARY_FOR_DISTRIBUTION=YES

    - name: Build Framework for Simulator (x86_64 & arm64)
      run: |
        cd GeometrizeFramework
        xcodebuild -scheme GeometrizeFramework \
          -configuration Release \
          -arch x86_64 \
          -sdk iphonesimulator \
          -derivedDataPath build_simulator \
          BUILD_LIBRARY_FOR_DISTRIBUTION=YES

    - name: Create XCFramework
      run: |
        xcodebuild -create-xcframework \
          -framework GeometrizeFramework/build_device/Release-iphoneos/GeometrizeFramework.framework \
          -framework GeometrizeFramework/build_simulator/Release-iphonesimulator/GeometrizeFramework.framework \
          -output GeometrizeFramework.xcframework

    - name: Compress Framework
      run: |
        zip -r GeometrizeFramework.xcframework.zip GeometrizeFramework.xcframework

    - name: Upload Framework Artifact
      uses: actions/upload-artifact@v3
      with:
        name: GeometrizeFramework
        path: GeometrizeFramework.xcframework.zip
        retention-days: 30

    - name: Create Release
      if: startsWith(github.ref, 'refs/tags/v')
      uses: softprops/action-gh-release@v1
      with:
        files: GeometrizeFramework.xcframework.zip
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
