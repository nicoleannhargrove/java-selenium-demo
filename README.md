# java-selenium-demo

Runs Selenium Web UI tests using Java, Gradle, WebDriverManager, and GitHub actions

### Part 0: Setting up the Java Project

- File -> New Project
- Choose project name, directory
- Under `dependencies` section in `build.gradle` add:
    - ```java
        testImplementation 'junit:junit:4.13.2'
        implementation 'org.seleniumhq.selenium:selenium-java:4.4.0'
        ```
    - Click on "Load Gradle Changes" (Shift + Command + I)

### Part 1: Installing ChromeDriver Manually

- We will need to create an instance of a web driver; we can use ChromeDriver for now
    - Go to https://chromedriver.chromium.org/downloads to download the appropriate driver for your machine
        - Unzip the file and move it to `/src/main/resources`
        - ⚠️ **Note:** In macOS, you may receive `error: chromedriver cannot be opened because the developer cannot be verified.` We will need to make MacOS trust  `chromedriver`  binary. This can be done in two steps:
            - Find the directory of `chromedriver`
            - Tell OS to trust the binary by lifting the quarantine:
                - `xattr -d com.apple.quarantine /usr/local/bin/chromedriver`
                - Replace `/usr/local/bin` with wherever your driver is
            - via [Fixing error: “chromedriver” cannot be opened because the developer cannot be verified. Unable to launch the chrome browser on Mac OS ⚡](https://timonweb.com/misc/fixing-error-chromedriver-cannot-be-opened-because-the-developer-cannot-be-verified-unable-to-launch-the-chrome-browser-on-mac-os/) via TimOnWeb
- Create a test class file `TestImplementation.java` in `/src/test/java`
    - ```java
        import org.junit.*;
        import org.openqa.selenium.WebDriver;
        import org.openqa.selenium.chrome.ChromeDriver;
        
        import static org.hamcrest.CoreMatchers.containsString;
        import static org.hamcrest.MatcherAssert.assertThat;
        
        public class TestImplementation {
            private WebDriver driver;
            @BeforeClass
            public static void setupWebdriverChromeDriver() {
                System.setProperty("webdriver.chrome.driver", System.getProperty("user.dir") + "/src/main/resources/chromedriver");
            }
        
            @Before
            public void setup() {
                driver = new ChromeDriver();
            }
        
            @After
            public void teardown() {
                if (driver != null) {
                    driver.quit();
                }
            }
        
            @Test
            public void verifyGitHubTitle() {
                driver.get("https://www.github.com");
                assertThat(driver.getTitle(), containsString("GitHub"));
            }
        }
        ```
- Code will work but you'll get `SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".`
    - Go to https://mvnrepository.com/artifact/org.slf4j/slf4j-simple/1.6.1 and add Gradle implementation:
        - ```java
            // https://mvnrepository.com/artifact/org.slf4j/slf4j-simple
            testImplementation group: 'org.slf4j', name: 'slf4j-simple', version: '1.6.1'
            
            ```
- Run the test
    - You can do this via IntelliJ or via CLI: `gradle test`
- You can view HTML reports at the command line via `open build/reports/tests/test/index.html`

### Part 2: Using `WebDriverManager`

- This time we'll use [[WebDriverManager]] to automate the browser setup
- After Part 1, add the following `build.gradle` dependency:
    - `testImplementation group: 'io.github.bonigarcia', name: 'webdrivermanager', version: '3.3.0'`
- In the test file, change `@BeforeClass` and `@Before` appropriately:
    - ```java
        // ...
        import org.openqa.selenium.chrome.ChromeDriver;
        // import org.openqa.selenium.firefox.FirefoxDriver;
        // ...
        
        @BeforeClass
        public static void setupWebdriverBinary() {
        WebDriverManager.chromedriver().setup();
        // WebDriverManager.firefoxdriver().setup();
        }
        @Before
        public void setup() {
        driver = new ChromeDriver();
        // driver = new FirefoxDriver();
        }
        ```
- To prove we're not using the local ChromeDriver we previously added in Part 1, let's remove the `chromedriver` file and re-run our tests:
    - `rm src/main/resources/chromedriver`
- See `using-webdriver` branch on https://github.com/ErnieAtLYD/java-selenium-exercise
#### Resources

- [WebDriverManager](https://bonigarcia.dev/webdrivermanager/)
- [Automating management of Selenium webdriver binaries with Java](https://medium.com/agency04/automating-management-of-selenium-webdriver-binaries-with-java-3a70c4a9b2d5) by Hrvoje Ruhek on Medium
- [How to manage browsers binaries using WebDriverManager in Selenium?](https://www.toolsqa.com/selenium-webdriver/webdrivermanager/)

### Part 3: Moving all of this to GitHub Actions
- Let's try having a CI/CD such as GitHub Actions do the work for us
    - We don't have to worry about ChromeDriver executables now that WebDriverManager is handling this
    - But we DO have to assume that the Linux machine that GitHub actions will use won't have Chrome installed. We also have to figure out how to do this all in headless mode
        - Headless mode operates as your typical browser would, but without a user interface, making it excellent for automated testing
            - Source: [Why Should I Run My Selenium Tests in Headless?](https://smartbear.com/blog/selenium-tests-headless/)
- Go into headless mode
    - add `import org.openqa.selenium.chrome.ChromeOptions;`
    - Change `setup()` to the following:
        - ```java
            @Before
            public void setup() {
                ChromeOptions options = new ChromeOptions();
                options.addArguments("--no-sandbox");
                options.addArguments("--disable-dev-shm-usage");
                options.addArguments("--headless");
                driver = new ChromeDriver(options);
                // driver = new FirefoxDriver();
            }
            ```
- Write a bash script called `InstallChrome.sh` that will install
    - ```bash
        #!/bin/bash
        set -ex
        wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
        sudo apt install ./google-chrome-stable_current_amd64.deb
        ```
- Create the GitHub action workflow
    - Create `.github/workflows/gradle.yml`:
        - ```yml
            name: Selenium Java CI
            
            on: [push]
            
            jobs:
            build:
                runs-on: ubuntu-latest # Using linux machine
            
                steps:
                - uses: actions/checkout@v2 # Checkout the code
                - name: Set up JDK 1.8
                    uses: actions/setup-java@v1 # Setup JAVA
                    with:
                    java-version: 1.8
                - name: Install Google Chrome # Using shell script to install Google Chrome
                    run: |
                    chmod +x ./scripts/InstallChrome.sh
                        ./scripts/InstallChrome.sh
                - name: Grant execute permission for gradlew
                    run: chmod +x gradlew # give permission to Gradle to run the commands
                - name: Build with Gradle
                    run: ./gradlew test --info # Run our tests using Gradle
            
            ```
    - Submit a pull request
- See `using-github-actions` branch on https://github.com/ErnieAtLYD/java-selenium-exercise
#### Resources

- [Running Selenium Web Tests with GitHub Actions](https://www.linkedin.com/pulse/running-selenium-web-tests-github-actions-moataz-nabil/) by Moataz Nabil on LinkedIn
- [Executing Selenium Tests ( Maven + Java ) with CI GitHub Actions](https://medium.com/@saurabhdube/running-selenium-web-tests-maven-java-with-github-actions-a20cba622af4) by Saurabh Dubey on Medium
- [How to setup Selenium Java with Gradle in IntelliJ](https://medium.com/@milosz.wozniak/how-to-setup-selenium-java-with-gradle-in-intellij-6c8b982dbb21) by Milosz Wozniak on Medium
- https://github.com/miosz/intellij-selenium-java
- [ChromeDriver - WebDriver for Chrome - Getting started](https://chromedriver.chromium.org/getting-started)
