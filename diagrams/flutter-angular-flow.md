# Flutter + Angular Integration Flow

```mermaid
flowchart LR
    subgraph Browser["One browser tab"]
        direction TB
        subgraph FL["Flutter web app (http://localhost:8080)"]
            Hdr["Header + top navigation (PortalShell)"]
            Content["Content area"]
            View["AngularModuleView<br/>registers an iframe platform view"]
        end
        subgraph NG["Angular app (http://localhost:4200) inside the iframe"]
            Boot["App component<br/>AuthService.connect()"]
            Router["Angular router<br/>/travel, /liability, ..."]
            Page["QuoteForm / QuoteResult"]
        end
        Content --> View
        View -. "iframe src" .-> NG
        Boot -. "postMessage il-ready" .-> View
        View -. "postMessage il-handoff + one-time token" .-> Boot
    end
    NG -->|"JWT"| API[ILAPI]
    FL -->|"JWT"| API
```

## What Flutter shows for each menu item

```mermaid
flowchart TD
    Menu["Flutter menu / route"] --> M{which route?}
    M -- "/dashboard" --> DashboardHome
    M -- "/motor" --> MotorScreen["MotorScreen -> QuoteFormView('motor')"]
    M -- "/home" --> HomeScreen["HomeScreen -> QuoteFormView('home')"]
    M -- "/travel" --> TravelScreen["TravelScreen -> AngularModuleView('travel')"]
    M -- "/liability" --> LiabilityScreen["LiabilityScreen -> AngularModuleView('liability')"]
    M -- "/customers /quotes /users /roles /reports /underwriting /profile /my-quotes" --> DataListView
    M -- "/policies" --> Placeholder["placeholder text (no data yet)"]
```

Angular never renders a header, navigation, dashboard, login or logout; it renders only the quote form or result inside the iframe.
