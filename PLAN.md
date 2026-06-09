# Build: Origami Craft Assistant

## Context
The Origami Craft Assistant helps paper craft enthusiasts discover and create origami projects tailored to their interests and available materials. Users photograph their paper supplies, and the app uses on-device AI to suggest appropriate origami patterns, provides real-time folding guidance through camera analysis, and connects crafters through social meetup planning. The app targets hobbyists and educators who want personalized craft recommendations and community connections. This idea showcases Apple's Foundation Models framework for on-device AI processing, enabling privacy-preserving analysis of materials and real-time folding assistance.

## Tech Stack
- **SwiftUI** - Native iOS UI following Apple HIG with SF Symbols
- **Foundation Models** - On-device AI for material recognition and folding guidance
- **Vision Framework** - Image analysis for paper detection and fold verification
- **Core ML** - Custom models for origami pattern matching
- **SwiftData** - Persistent storage for projects, materials, and user preferences
- **PhotosUI** - Material inventory photo management
- **AVFoundation** - Real-time camera feed for folding assistance
- **CloudKit** - Sync user projects and enable social features
- **MapKit** - Meetup location planning and discovery
- **StoreKit 2** - Premium pattern packs and subscription features

## Phase 1: Project Setup & Scaffolding

Create Xcode project:
```yaml
# project.yml for xcodegen
name: OrigamiAssistant
options:
  bundleIdPrefix: com.origamicraft
  deploymentTarget:
    iOS: "18.0"
targets:
  OrigamiAssistant:
    type: application
    platform: iOS
    sources: [OrigamiAssistant]
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.origamicraft.assistant
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
        INFOPLIST_KEY_UISupportedInterfaceOrientations: [UIInterfaceOrientationPortrait, UIInterfaceOrientationLandscapeLeft, UIInterfaceOrientationLandscapeRight]
    capabilities:
      - com.apple.developer.icloud-container-identifiers
      - com.apple.developer.icloud-services
      - com.apple.developer.in-app-purchases
    entitlements:
      com.apple.developer.icloud-container-identifiers: [iCloud.com.origamicraft.assistant]
      com.apple.developer.icloud-services: [CloudKit]
      com.apple.security.app-sandbox: true
      com.apple.security.device.camera: true
      com.apple.security.personal-information.photos-library: true
      com.apple.security.personal-information.location: true
```

Required capabilities:
- Camera Usage Description: "Origami Assistant needs camera access to analyze your paper materials and guide your folding"
- Photo Library Usage Description: "Save photos of your paper materials to get personalized project suggestions"
- Location When In Use Usage Description: "Find and plan origami meetups near you"

Core data models:
```swift
import Foundation
import SwiftData
import CoreLocation

@Model
final class PaperMaterial {
    var id: UUID
    var color: String
    var size: PaperSize
    var texture: String
    var thickness: Double // in mm
    var quantity: Int
    var photoData: Data?
    var dateAdded: Date
    
    init(color: String, size: PaperSize, texture: String = "smooth", thickness: Double = 0.1, quantity: Int = 1) {
        self.id = UUID()
        self.color = color
        self.size = size
        self.texture = texture
        self.thickness = thickness
        self.quantity = quantity
        self.dateAdded = Date()
    }
}

enum PaperSize: String, Codable, CaseIterable {
    case square6 = "6x6"
    case square15 = "15x15"
    case square20 = "20x20"
    case a4 = "A4"
    case letter = "Letter"
}

@Model
final class OrigamiProject {
    var id: UUID
    var name: String
    var difficulty: Difficulty
    var estimatedTime: Int // minutes
    var requiredPaperSize: PaperSize
    var instructions: [FoldingStep]
    var tags: [String]
    var completedCount: Int
    var isFavorite: Bool
    var lastAttempted: Date?
    
    enum Difficulty: String, Codable, CaseIterable {
        case beginner, intermediate, advanced, expert
    }
}

struct FoldingStep: Codable {
    let id: UUID
    let instruction: String
    let diagramImageName: String?
    let validationPoints: [CGPoint] // Key points to check via Vision
}

@Model
final class Meetup {
    var id: UUID
    var title: String
    var date: Date
    var location: CLLocationCoordinate2D
    var locationName: String
    var hostUserId: String
    var participantIds: [String]
    var projectId: UUID?
    var maxParticipants: Int
    
    var coordinate: CLLocationCoordinate2D {
        get { location }
        set { location = newValue }
    }
}
```

Sample data:
```swift
extension PaperMaterial {
    static let sampleMaterials = [
        PaperMaterial(color: "Red", size: .square15, texture: "smooth", quantity: 10),
        PaperMaterial(color: "Blue", size: .square20, texture: "textured", thickness: 0.15, quantity: 5),
        PaperMaterial(color: "Gold", size: .square6, texture: "metallic", thickness: 0.12, quantity: 20),
        PaperMaterial(color: "White", size: .a4, texture: "smooth", quantity: 50)
    ]
}

extension OrigamiProject {
    static let sampleProjects = [
        OrigamiProject(
            name: "Classic Crane",
            difficulty: .beginner,
            estimatedTime: 10,
            requiredPaperSize: .square15,
            tags: ["traditional", "animals", "lucky"]
        ),
        OrigamiProject(
            name: "Kawasaki Rose",
            difficulty: .expert,
            estimatedTime: 45,
            requiredPaperSize: .square20,
            tags: ["flowers", "decorative", "complex"]
        )
    ]
}
```

## Phase 2: Core Data Model & Backend

SwiftData container setup:
```swift
import SwiftData

@MainActor
final class DataController {
    static let shared = DataController()
    
    let container: ModelContainer
    
    init() {
        let schema = Schema([
            PaperMaterial.self,
            OrigamiProject.self,
            Meetup.self
        ])
        
        let modelConfiguration = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: false,
            cloudKitDatabase: .automatic
        )
        
        do {
            container = try ModelContainer(
                for: schema,
                configurations: [modelConfiguration]
            )
        } catch {
            fatalError("Could not create ModelContainer: \(error)")
        }
    }
}
```

AI analysis service using Foundation Models:
```swift
import Foundation
import CoreML
import Vision

@MainActor
final class AIAnalysisService: ObservableObject {
    @Published var isAnalyzing = false
    @Published var lastAnalysisResult: AnalysisResult?
    
    struct AnalysisResult {
        let detectedColors: [String]
        let paperSize: PaperSize?
        let suggestedProjects: [OrigamiProject]
        let confidence: Double
    }
    
    func analyzePaperPhoto(_ imageData: Data) async throws -> AnalysisResult {
        // Foundation Models integration for paper analysis
        // Returns detected paper properties and project suggestions
    }
    
    func validateFold(currentImage: CGImage, targetStep: FoldingStep) async throws -> FoldValidation {
        // Vision framework + Foundation Models for fold verification
        // Returns accuracy score and correction suggestions
    }
}

struct FoldValidation {
    let isCorrect: Bool
    let accuracy: Double
    let suggestions: [String]
    let highlightRegions: [CGRect]
}
```

## Phase 3: UI & Views

Main tab view structure:
```swift
struct ContentView: View {
    @State private var selectedTab = 0
    
    var body: some View {
        TabView(selection: $selectedTab) {
            MaterialsInventoryView()
                .tabItem {
                    Label("Materials", systemImage: "square.stack.3d.up")
                }
                .tag(0)
            
            ProjectBrowserView()
                .tabItem {
                    Label("Projects", systemImage: "sparkles.rectangle.stack")
                }
                .tag(1)
            
            FoldingAssistantView()
                .tabItem {
                    Label("Fold", systemImage: "camera.viewfinder")
                }
                .tag(2)
            
            MeetupMapView()
                .tabItem {
                    Label("Meetups", systemImage: "mappin.and.ellipse")
                }
                .tag(3)
            
            ProfileView()
                .tabItem {
                    Label("Profile", systemImage: "person.circle")
                }
                .tag(4)
        }
    }
}
```

Materials inventory view - displays user's paper collection with AI-powered addition:
```swift
struct MaterialsInventoryView: View {
    @Query private var materials: [PaperMaterial]
    @StateObject private var aiService = AIAnalysisService()
    @State private var showingPhotoPicker = false
    
    // Grid display of materials with photo capture and AI analysis
    // Binds to materials query and aiService published properties
}
```

Project browser view - AI-recommended projects based on available materials:
```swift
struct ProjectBrowserView: View {
    @Query private var projects: [OrigamiProject]
    @Query private var materials: [PaperMaterial]
    @State private var selectedDifficulty: OrigamiProject.Difficulty?
    
    // Filtered project list with AI recommendations highlighted
    // Binds to projects and materials queries
}
```

Folding assistant view - real-time camera guidance:
```swift
struct FoldingAssistantView: View {
    @State private var currentProject: OrigamiProject?
    @State private var currentStepIndex = 0
    @StateObject private var cameraManager = CameraManager()
    
    // Camera preview with overlay guidance
    // Binds to cameraManager for real-time analysis
}
```

## Phase 4: Feature Implementation

**Material Recognition Feature:**
- User captures photo of paper materials
- Foundation Models analyzes color, size, and texture
- Automatically creates PaperMaterial entries with detected properties
- Handles multiple papers in single photo
- Edge cases: poor lighting, partial visibility, non-paper objects

**Project Recommendation Engine:**
- Analyzes user's material inventory
- Uses Foundation Models to match materials to suitable projects
- Considers user's skill progression and preferences
- Updates recommendations as inventory changes
- Provides alternative material suggestions for desired projects

**Real-time Folding Guidance:**
- Camera feed with step-by-step overlay instructions
- Vision framework detects paper edges and fold lines
- Foundation Models validates fold accuracy against expected result
- Provides corrective feedback through visual highlights and text
- Tracks completion progress and time spent

**Social Meetup Planning:**
- MapKit integration for location-based meetup discovery
- CloudKit sync for cross-device meetup data
- In-app meetup creation with project selection
- Participant limit management and RSVP system
- Push notifications for meetup reminders (requires remote notification capability)

**Premium Features via StoreKit 2:**
- Advanced project packs (complex models, seasonal collections)
- Unlimited AI analyses per month
- Priority meetup hosting
- Handles purchase restoration and subscription management

## Phase 5: Polish & Integration

**Adaptive Layout:**
- Responsive grid for materials inventory adjusting to screen size
- iPad support with sidebar navigation in regular width
- Dynamic Type support for all text elements
- Dark mode optimized color schemes for paper visibility

**Loading States:**
- Skeleton views during AI analysis
- Progress indicators for multi-step operations
- Graceful degradation when AI features unavailable

**Error Handling:**
- Network connectivity issues for CloudKit sync
- Camera permission denials with clear recovery flow
- AI model loading failures with offline fallbacks

**Visual Polish:**
- Custom paper texture overlays in UI
- Smooth transitions between folding steps
- Haptic feedback for successful fold validation
- Particle effects for project completion

**App Intents:**
- "Start folding [project name]" Siri shortcut
- "Show my paper inventory" intent
- "Find origami meetups near me" with location parameter

**Widgets:**
- Project of the day widget
- Material inventory summary
- Upcoming meetup countdown

## Verification

Build and run commands:
```bash
# Generate Xcode project
xcodegen generate

# Build for simulator
xcodebuild -scheme OrigamiAssistant -destination 'platform=iOS Simulator,name=iPhone 15 Pro' build

# Run on simulator
xcrun simctl boot "iPhone 15 Pro"
open -a Simulator
xcrun simctl install booted build/Debug-iphonesimulator/OrigamiAssistant.app
xcrun simctl launch booted com.origamicraft.assistant
```

Key user flows to test:
1. **First Launch:** Onboarding → Camera permissions → Add first material → Get project recommendations
2. **Complete Project:** Select project → Start folding assistant → Validate each step → Mark complete
3. **Plan Meetup:** Browse map → Create meetup → Invite participants → Receive confirmations
4. **Premium Upgrade:** Browse locked content → Purchase subscription → Verify access → Restore on second device

Edge cases to verify:
- AI analysis with unusual paper colors/patterns
- Folding validation in low light conditions
- Offline mode with cached projects
- CloudKit sync conflicts resolution
- Subscription expiration handling

Deployment requirements:
- Apple Developer account for CloudKit container and StoreKit testing
- iOS 18.0+ for Foundation Models framework
- Physical device recommended for camera-based features
- TestFlight for beta testing meetup features