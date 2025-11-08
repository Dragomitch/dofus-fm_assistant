# Section 5: UI/Display Layer Migration

## 5.1 Current Python UI - Tkinter Implementation

### File: `display.py`

The current application uses **Tkinter**, Python's standard GUI library, running in a separate thread.

### Architecture Overview

```python
class Display(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.start()  # Start GUI thread

    def run(self):
        # Create Tkinter window and widgets
        self.root = Tk()
        self.root.title("FM Helper")
        # ... create widgets
        self.root.mainloop()  # Run event loop
```

**Key Points**:
- GUI runs in **separate thread** from packet sniffer
- Uses **observer pattern**: Item/Rune objects call `listener.updateItem()` to refresh UI
- **Synchronous updates**: Main thread calls GUI update methods

---

## 5.2 UI Layout and Components

### Current UI Structure

```
┌────────────────────────────────────────────────────────┐
│                   FM Helper                            │
├────────────────────────────────────────────────────────┤
│ Item :  [Gelano] (niveau : 199)                       │
│ Rune :  [Ra Vi] | +10 Vitalité (poids : 2.5)          │
├────────────────────────────────────────────────────────┤
│ Min  Max  Effet              Modif  Poids             │
│ 30   40   +35 Vitalité       +5     8.75/10.0         │
│ 20   30   +25 Force          +0     25.0/30.0         │
│ 10   20   +15 Agilité        -2     13.0/20.0  ◄ RED  │
│ 1    1    +1 PA              +0     100.0/100.0       │
├────────────────────────────────────────────────────────┤
│ Reliquat : 12.5 (+2.0)                                │
└────────────────────────────────────────────────────────┘
```

### UI Components Breakdown

#### 1. **Header Section**
- **Item Name & Level**: `[Gelano] (niveau : 199)`
- **Rune Info**: `[Ra Vi] | +10 Vitalité (poids : 2.5)`

#### 2. **Stat Lines Table**
Columns:
- **Min**: Minimum possible value for stat
- **Max**: Maximum possible value for stat
- **Effet**: Human-readable stat description with current value
- **Modif**: Change from last FM attempt (+5, -2, etc.)
- **Poids**: Current weight / Max weight (e.g., `8.75/10.0`)

**Color Coding**:
- **Green**: Stat increased (`last_modification > 0`)
- **Red**: Stat decreased (`last_modification < 0`)
- **Default**: No change (`last_modification == 0`)

#### 3. **Reliquat Display**
- **Format**: `Reliquat : 12.5 (+2.0)`
  - `12.5` = current total reliquat
  - `(+2.0)` = change from last FM attempt
- Shows with `+` or `-` sign for clarity

---

## 5.3 Python Implementation Details

### Main Window Creation

```python
def run(self):
    self.root = Tk()
    self.root.title("FM Helper")
    self.root.protocol("WM_DELETE_WINDOW", self.close)

    # Main container
    self.mainframe = ttk.Frame(self.root, padding="3 3 12 12")
    self.mainframe.grid(column=0, row=0, sticky=(N, W, E, S))

    # Item label
    ttk.Label(self.mainframe, text="Item :").grid(column=1, row=1, sticky=E)
    self.item = ttk.Label(self.mainframe, text="no item")
    self.item.grid(column=2, row=1, sticky=W)

    # Rune label
    ttk.Label(self.mainframe, text="Rune :").grid(column=1, row=2, sticky=E)
    self.rune = ttk.Label(self.mainframe, text="no rune")
    self.rune.grid(column=2, row=2, sticky=W)

    # Stat lines frame (dynamic)
    self.lines = ttk.Frame(self.mainframe, padding="3 3 12 12")
    self.lines.grid(column=1, columnspan=2, row=3, sticky=E+W)

    # Reliquat label
    ttk.Label(self.mainframe, text="Reliquat :").grid(column=1, row=4, sticky=E)
    self.reliquat = ttk.Label(self.mainframe, text="aucun")
    self.reliquat.grid(column=2, row=4, sticky=W)

    self.root.mainloop()
```

---

### Update Methods

#### Update Rune Display

```python
def updateRune(self, rune):
    self.rune["text"] = (
        rune.getName() + ' | ' +
        rune.getDescription() + ' (poids : ' +
        str(rune.getWeight()) + ')'
    )
```

**Example Output**: `Ra Vi | +10 Vitalité (poids : 2.5)`

---

#### Update Item Display

```python
def updateItem(self, item):
    # Update item name and level
    self.item["text"] = item.getName() + ' (niveau : ' + str(item.getLevel()) + ')'

    # Update reliquat
    self.reliquat["text"] = self.myStr(item.getReliquat())
    if item.getLastReliquatModification() != 0:
        self.reliquat["text"] += ' (' + self.myStrWithSign(item.getLastReliquatModification()) + ')'

    # Clear existing stat line widgets (except header)
    for widget in self.lines.winfo_children():
        if widget.grid_info()["row"] != 1:
            widget.destroy()

    # Create new stat line widgets
    row = 2
    for line in item.getLines():
        ttk.Label(self.lines, text=str(line.getMin())).grid(column=1, row=row, sticky=W)
        ttk.Label(self.lines, text=str(line.getMax())).grid(column=2, row=row, sticky=W)
        ttk.Label(self.lines, text=line.getDescription()).grid(column=3, row=row, sticky=W)
        ttk.Label(self.lines, text=self.myStrWithSign(line.getLastModification())).grid(column=4, row=row, sticky=W)
        ttk.Label(self.lines, text=self.myStr(line.getWeight()) + "/" + self.myStr(line.getMaxWeight())).grid(column=5, row=row, sticky=W)

        # Apply color based on modification
        if line.getLastModification() > 0:
            for widget in self.lines.winfo_children():
                if widget.grid_info()["row"] == row:
                    widget['foreground'] = 'green'
        elif line.getLastModification() < 0:
            for widget in self.lines.winfo_children():
                if widget.grid_info()["row"] == row:
                    widget['foreground'] = 'red'

        row += 1
```

**Key Logic**:
1. Destroy all stat line widgets except header
2. Recreate all widgets from current item state
3. Apply color coding based on `last_modification`

---

### Utility Methods

```python
def myStr(self, number):
    """Format number: remove decimal if integer"""
    if number == int(number):
        return str(int(number))
    else:
        return "%.1f" % number

def myStrWithSign(self, number):
    """Format number with + or - sign"""
    if number > 0:
        return ('+' + str(number))
    else:
        return str(number)
```

---

## 5.4 Java GUI Options

### Option 1: JavaFX (Recommended)

**Pros**:
- Modern, native Java GUI framework
- Rich component library
- FXML for declarative UI
- Good CSS styling support
- Active development

**Cons**:
- Requires separate dependency (not in JDK by default since Java 11)
- Learning curve if unfamiliar

**Maven Dependency**:
```xml
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-controls</artifactId>
    <version>21.0.1</version>
</dependency>
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-fxml</artifactId>
    <version>21.0.1</version>
</dependency>
```

---

### Option 2: Swing (Not Recommended)

**Pros**:
- Built into JDK
- Mature, stable

**Cons**:
- Outdated look and feel
- Less modern than JavaFX
- Verbose API

---

### Option 3: Web UI (Spring Boot + Thymeleaf/React)

**Pros**:
- Modern web tech stack
- Easy to style with CSS
- Can add remote access capability
- Reactive updates with WebSockets

**Cons**:
- More complex architecture
- Requires browser
- Overkill for local desktop app

---

### Recommended: JavaFX

For a 1:1 migration maintaining desktop app nature, **JavaFX** is the best choice.

---

## 5.5 JavaFX Implementation

### Project Structure

```
com.dofus.fm.ui
├── FMApplication.java          // JavaFX Application entry point
├── controller
│   └── MainController.java     // FXML controller
├── service
│   └── DisplayService.java     // Business logic → UI bridge
└── fxml
    └── main.fxml               // UI layout definition
```

---

### JavaFX Application Entry Point

```java
package com.dofus.fm.ui;

import javafx.application.Application;
import javafx.fxml.FXMLLoader;
import javafx.scene.Scene;
import javafx.stage.Stage;
import org.springframework.boot.SpringApplication;
import org.springframework.context.ConfigurableApplicationContext;

public class FMApplication extends Application {

    private ConfigurableApplicationContext springContext;

    @Override
    public void init() {
        // Initialize Spring Boot context
        springContext = SpringApplication.run(DofusFMAssistantApplication.class);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {
        // Load FXML with Spring integration
        FXMLLoader loader = new FXMLLoader(getClass().getResource("/fxml/main.fxml"));
        loader.setControllerFactory(springContext::getBean);

        Scene scene = new Scene(loader.load(), 600, 400);
        primaryStage.setTitle("FM Helper");
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    @Override
    public void stop() {
        springContext.close();
    }

    public static void main(String[] args) {
        launch(args);
    }
}
```

---

### FXML Layout Definition

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>
<?import javafx.geometry.Insets?>

<VBox xmlns:fx="http://javafx.com/fxml"
      fx:controller="com.dofus.fm.ui.controller.MainController"
      spacing="10" padding="10">

    <!-- Item Info -->
    <HBox spacing="10">
        <Label text="Item :" styleClass="label-header"/>
        <Label fx:id="itemLabel" text="no item"/>
    </HBox>

    <!-- Rune Info -->
    <HBox spacing="10">
        <Label text="Rune :" styleClass="label-header"/>
        <Label fx:id="runeLabel" text="no rune"/>
    </HBox>

    <!-- Stat Lines Table -->
    <TableView fx:id="statLinesTable" VBox.vgrow="ALWAYS">
        <columns>
            <TableColumn fx:id="minColumn" text="Min" prefWidth="50"/>
            <TableColumn fx:id="maxColumn" text="Max" prefWidth="50"/>
            <TableColumn fx:id="effectColumn" text="Effet" prefWidth="200"/>
            <TableColumn fx:id="modifColumn" text="Modif" prefWidth="60"/>
            <TableColumn fx:id="weightColumn" text="Poids" prefWidth="100"/>
        </columns>
    </TableView>

    <!-- Reliquat -->
    <HBox spacing="10">
        <Label text="Reliquat :" styleClass="label-header"/>
        <Label fx:id="reliquatLabel" text="aucun"/>
    </HBox>

</VBox>
```

---

### MainController (FXML Controller)

```java
package com.dofus.fm.ui.controller;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Line;
import com.dofus.fm.domain.Rune;
import javafx.application.Platform;
import javafx.collections.FXCollections;
import javafx.collections.ObservableList;
import javafx.fxml.FXML;
import javafx.scene.control.*;
import javafx.scene.paint.Color;
import org.springframework.stereotype.Component;

@Component
public class MainController {

    @FXML private Label itemLabel;
    @FXML private Label runeLabel;
    @FXML private Label reliquatLabel;

    @FXML private TableView<LineViewModel> statLinesTable;
    @FXML private TableColumn<LineViewModel, String> minColumn;
    @FXML private TableColumn<LineViewModel, String> maxColumn;
    @FXML private TableColumn<LineViewModel, String> effectColumn;
    @FXML private TableColumn<LineViewModel, String> modifColumn;
    @FXML private TableColumn<LineViewModel, String> weightColumn;

    private ObservableList<LineViewModel> statLines = FXCollections.observableArrayList();

    @FXML
    public void initialize() {
        // Bind table columns
        minColumn.setCellValueFactory(cell -> cell.getValue().minProperty());
        maxColumn.setCellValueFactory(cell -> cell.getValue().maxProperty());
        effectColumn.setCellValueFactory(cell -> cell.getValue().descriptionProperty());
        modifColumn.setCellValueFactory(cell -> cell.getValue().modificationProperty());
        weightColumn.setCellValueFactory(cell -> cell.getValue().weightProperty());

        // Apply row coloring
        statLinesTable.setRowFactory(tv -> new TableRow<LineViewModel>() {
            @Override
            protected void updateItem(LineViewModel item, boolean empty) {
                super.updateItem(item, empty);
                if (item == null || empty) {
                    setStyle("");
                } else if (item.getModificationValue() > 0) {
                    setStyle("-fx-text-fill: green;");
                } else if (item.getModificationValue() < 0) {
                    setStyle("-fx-text-fill: red;");
                } else {
                    setStyle("");
                }
            }
        });

        statLinesTable.setItems(statLines);
    }

    /**
     * Update rune display (called from DisplayService)
     */
    public void updateRune(Rune rune) {
        Platform.runLater(() -> {
            String runeText = String.format("%s | %s (poids : %d)",
                rune.getName(),
                rune.getDescription(),
                rune.getWeight()
            );
            runeLabel.setText(runeText);
        });
    }

    /**
     * Update item display (called from DisplayService)
     */
    public void updateItem(Item item) {
        Platform.runLater(() -> {
            // Update item label
            String itemText = String.format("%s (niveau : %d)",
                item.getName(),
                item.getLevel()
            );
            itemLabel.setText(itemText);

            // Update reliquat
            String reliquatText = formatNumber(item.getReliquat());
            if (item.getLastReliquatModification() != 0) {
                reliquatText += " (" + formatNumberWithSign(item.getLastReliquatModification()) + ")";
            }
            reliquatLabel.setText(reliquatText);

            // Update stat lines table
            statLines.clear();
            for (Line line : item.getLines()) {
                statLines.add(new LineViewModel(line));
            }
        });
    }

    private String formatNumber(double number) {
        if (number == (int) number) {
            return String.valueOf((int) number);
        } else {
            return String.format("%.1f", number);
        }
    }

    private String formatNumberWithSign(double number) {
        if (number > 0) {
            return "+" + formatNumber(number);
        } else {
            return formatNumber(number);
        }
    }
}
```

---

### LineViewModel (Table Row Data)

```java
package com.dofus.fm.ui.controller;

import com.dofus.fm.domain.Line;
import javafx.beans.property.SimpleStringProperty;
import javafx.beans.property.StringProperty;

/**
 * ViewModel for displaying a stat line in the table
 */
public class LineViewModel {

    private final Line line;

    private final StringProperty min;
    private final StringProperty max;
    private final StringProperty description;
    private final StringProperty modification;
    private final StringProperty weight;

    public LineViewModel(Line line) {
        this.line = line;

        this.min = new SimpleStringProperty(String.valueOf(line.getMin()));
        this.max = new SimpleStringProperty(String.valueOf(line.getMax()));
        this.description = new SimpleStringProperty(line.getDescription());

        String modifStr = line.getLastModification() > 0
            ? "+" + line.getLastModification()
            : String.valueOf(line.getLastModification());
        this.modification = new SimpleStringProperty(modifStr);

        String weightStr = formatNumber(line.getWeight()) + "/" + formatNumber(line.getMaxWeight());
        this.weight = new SimpleStringProperty(weightStr);
    }

    public StringProperty minProperty() { return min; }
    public StringProperty maxProperty() { return max; }
    public StringProperty descriptionProperty() { return description; }
    public StringProperty modificationProperty() { return modification; }
    public StringProperty weightProperty() { return weight; }

    public int getModificationValue() {
        return line.getLastModification();
    }

    private String formatNumber(double number) {
        if (number == (int) number) {
            return String.valueOf((int) number);
        } else {
            return String.format("%.1f", number);
        }
    }
}
```

---

### DisplayService (Bridge between Business Logic and UI)

```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Rune;
import com.dofus.fm.ui.controller.MainController;
import org.springframework.stereotype.Service;

/**
 * Service layer that bridges business logic and UI updates
 * Replaces Python's listener pattern
 */
@Service
public class DisplayService {

    private MainController mainController;

    /**
     * Register the UI controller (called after FXML loads)
     */
    public void setController(MainController controller) {
        this.mainController = controller;
    }

    /**
     * Update rune display
     */
    public void updateRune(Rune rune) {
        if (mainController != null) {
            mainController.updateRune(rune);
        }
    }

    /**
     * Update item display
     */
    public void updateItem(Item item) {
        if (mainController != null) {
            mainController.updateItem(item);
        }
    }
}
```

---

## 5.6 Threading Considerations

### Python Threading Model
```python
class Display(threading.Thread):
    # GUI runs in separate thread
```

### JavaFX Threading Model

**Critical**: JavaFX UI updates **must** occur on the JavaFX Application Thread!

```java
// WRONG - will crash!
public void updateItem(Item item) {
    itemLabel.setText(item.getName()); // Called from packet handler thread
}

// CORRECT - use Platform.runLater()
public void updateItem(Item item) {
    Platform.runLater(() -> {
        itemLabel.setText(item.getName());
    });
}
```

**Why**: JavaFX (like most GUI frameworks) is single-threaded. All UI updates must happen on the UI thread.

---

### Packet Handler → UI Update Flow

```
Packet Sniffer Thread
    ↓
DofusPacketParser
    ↓
FMSessionService.handleCraftResult()
    ↓
FMCalculationService.executeFM()
    ↓
DisplayService.updateItem()
    ↓
Platform.runLater(() -> { ... })  ← Switch to JavaFX thread
    ↓
MainController.updateItem()
    ↓
UI Updated
```

---

## 5.7 CSS Styling (Optional Enhancement)

JavaFX supports CSS for styling:

```css
/* styles.css */

.label-header {
    -fx-font-weight: bold;
    -fx-font-size: 14px;
}

.table-row-cell {
    -fx-font-family: monospace;
}

.positive-change {
    -fx-text-fill: green;
}

.negative-change {
    -fx-text-fill: red;
}
```

Load in FXML:
```xml
<VBox stylesheets="@../styles.css">
```

---

## 5.8 Alternative: Web UI with Spring Boot

### Architecture

```
Spring Boot Backend (REST API)
    ↓
WebSocket for real-time updates
    ↓
React/Vue.js Frontend
```

### Example REST Endpoint

```java
@RestController
@RequestMapping("/api/fm")
public class FMController {

    @Autowired
    private FMSessionService sessionService;

    @GetMapping("/current-item")
    public ItemDTO getCurrentItem() {
        return sessionService.getCurrentItem();
    }

    @GetMapping("/current-rune")
    public RuneDTO getCurrentRune() {
        return sessionService.getCurrentRune();
    }
}
```

### WebSocket for Real-Time Updates

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/fm-updates").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
    }
}

@Service
public class DisplayService {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    public void updateItem(Item item) {
        messagingTemplate.convertAndSend("/topic/item-update", item);
    }
}
```

### Frontend (React Example)

```jsx
import SockJS from 'sockjs-client';
import Stomp from 'stompjs';

const socket = new SockJS('/fm-updates');
const stompClient = Stomp.over(socket);

stompClient.connect({}, () => {
    stompClient.subscribe('/topic/item-update', (message) => {
        const item = JSON.parse(message.body);
        updateItemDisplay(item);
    });
});
```

**Pros**:
- Modern, responsive UI
- Can access from any device on network
- Easy to add charts/visualizations

**Cons**:
- More complex than desktop app
- Requires browser
- Additional dependencies

---

## 5.9 Recommended Approach

### For 1:1 Migration: **JavaFX**

**Reasons**:
1. Closest to original Tkinter desktop app
2. Native performance
3. Self-contained executable
4. No browser required
5. Simple architecture

### Implementation Steps:

1. **Setup JavaFX dependencies** (Maven/Gradle)
2. **Create FXML layout** matching Tkinter UI
3. **Implement MainController** with update methods
4. **Create DisplayService** to bridge business logic → UI
5. **Use Platform.runLater()** for thread-safe UI updates
6. **Test with real packets** to ensure updates work correctly

---

## 5.10 Key Migration Considerations

### 1. Thread Safety
- Always use `Platform.runLater()` for UI updates from non-UI threads
- Consider using `@Async` for packet processing to keep UI responsive

### 2. Data Binding
- Use JavaFX `ObservableList` for TableView
- Use `Property` types for reactive updates

### 3. Color Coding
- Implement custom row factory for TableView
- Apply styles based on `last_modification` value

### 4. Number Formatting
- Replicate Python's `myStr()` and `myStrWithSign()` helpers
- Consider using `DecimalFormat` for consistent formatting

### 5. Window Lifecycle
- Handle close button properly (`setOnCloseRequest`)
- Stop packet capture when window closes
- Clean up resources

### 6. Error Handling
- Show error dialogs for packet parsing failures
- Log exceptions for debugging

---

## 5.11 UI Testing Strategy

### Manual Testing Checklist
- [ ] Item display updates when packet received
- [ ] Rune display updates when packet received
- [ ] Stat lines show correct values
- [ ] Positive changes show in green
- [ ] Negative changes show in red
- [ ] Reliquat updates correctly
- [ ] Reliquat modification shows in parentheses
- [ ] Exotic lines appear/disappear correctly
- [ ] Window can be closed without errors

### Integration Testing
```java
@SpringBootTest
class DisplayServiceTest {

    @Autowired
    private DisplayService displayService;

    @Test
    void testUpdateItem_UIRefreshes() {
        // Create test item
        Item item = createTestItem();

        // Update display
        displayService.updateItem(item);

        // Verify UI updated (using TestFX)
        // ...
    }
}
```

### UI Testing with TestFX
```java
@ExtendWith(ApplicationExtension.class)
class MainControllerTest {

    @Test
    void testItemLabelUpdates(FxRobot robot) {
        Item item = createTestItem();
        item.setName("Test Item");
        item.setLevel(100);

        robot.interact(() -> {
            controller.updateItem(item);
        });

        Label itemLabel = robot.lookup("#itemLabel").query();
        assertEquals("Test Item (niveau : 100)", itemLabel.getText());
    }
}
```

---

**Next Section Preview:**
Section 6 will cover the complete Java/Spring Boot architecture, technology stack selection, and project structure.
