# eUmzug-Plattform (vor Aufgabe 4) (eumzug-plattform-2018-vor-aufgabe-4)

> Autoren der Dokumentation: Björn Scheppler

> Dokumentation letztmals aktualisiert: 26.9.2018

> TOC erstellt mit https://ecotrust-canada.github.io/markdown-toc/

In diesem Projekt ist eine mögliche Lösung für den [UmzugsmeldeProzess](https://www.egovernment.ch/de/umsetzung/schwerpunktplan/e-umzug-schweiz/) entwickelt.

Die Lösung entstand im Rahmen des **Moduls Geschäftsprozesssintegration im Studiengang Wirtschaftsinformatik** an der ZHAW School of Management and Law.

Dies ist das den Studierenden zur Verfügung gestellte Maven-Projekt für die Umzugsplattform als Ausgangslage, bevor sie Aufgabe 4 in Angriff nehmen.

## Inhaltsverzeichnis
  * [Inhaltsverzeichnis](#inhaltsverzeichnis)
  * [Architektur der Umzugsplattform inklusive Umsystemen](#architektur-der-umzugsplattform-inklusive-umsystemen)
    + [Haupt-Komponenten und ihr Zusammenspiel](#haupt-komponenten-und-ihr-zusammenspiel)
    + [Umzugsplattform](#umzugsplattform)
    + [Personenregister und weitere Umsyteme](#personenregister-und-weitere-umsyteme)
  * [(Technische) Komponenten der Umzugsplattform](#-technische--komponenten-der-umzugsplattform)
  * [Erforderliche Schritte für das Testen der Applikation](#erforderliche-schritte-f-r-das-testen-der-applikation)
    + [Voraussetzungen](#voraussetzungen)
    + [Deployment](#deployment)
  * [Nutzung (Testing) der Applikation](#nutzung--testing--der-applikation)
    + [Manuelles Testen](#manuelles-testen)
    + [Semi-Manuelles Testen](#semi-manuelles-testen)
    + [Automatisiertes Testen](#automatisiertes-testen)
  * [Prototypische Vereinfachungen](#prototypische-vereinfachungen)
  * [Mitwirkende](#mitwirkende)

## Architektur der Umzugsplattform inklusive Umsystemen

### Haupt-Komponenten und ihr Zusammenspiel
Später im Semster von den Studierenden zu erstellen.

### Umzugsplattform
Aufgeführt ist der im aktuellen Projekt umgesetzte Stand, nicht die fertige Lösung.

1. **Umzugsplattform insgesamt**: Der Einfachheit halber umfasst die Plattform alle folgende Komponenten in einer Applikation, die produktiv teilweise separat deployed würden:
    1. **Applikationsserver und Webserver**: Über Spring Boot wird automatisch ein In-Memory-Tomcat-Applikationsserver (inkl. Webserver) gestartet.
    2. **Process Engine**: Darin eingebettet wird die Camunda Process Engine als Applikation ausgeführt und ist per REST von aussen zugreifbar. Sie umfasst nebst der BPMN Core Engine auch einen Job Executor für das Erledigen asynchroner Aufgaben (z.B. Timer).
    3. **Process Engine-Datenbank**: Da eine Process Engine eine State Machine ist, muss sie irgendwo den Status laufender als auch abgeschlossener Prozessinstanzen persistieren können. Dies geschieht über eine File-basierte H2-Datenbank.
    5. **Tasklist-Applikation**: Damit der Meldepflichtige seine Meldung erfassen kann, wird eine Client-Applikation benötigt, die im Browser ausgeführt werden kann. Hierfür wird die ebenfalls im Applikations- und Webserver Tomcat eingebettete Camunda Tasklist WebApp eingesetzt. In einer produktiven Lösung würde diese sicher separat deployed oder sogar durch eine spezifisch für den Kanton Bern entwickelte Tasklist-Applikation ersetzt, welche über REST mit der Process Engine kommuniziert. Immerhin wurde die Tasklist-Webapp (und die übrigen Webapps) angepasst (deutsche Übersetzung, Logo und Farben, Google Maps API und Stripe Checkout integriert) in einem eigenen WebJAR-Projekt. Details hierzu siehe https://github.com/zhaw-gpi/be-services-plattform
    6. **Cockpit-Applikation**: Damit der Systemadministrator bei technischen Problemen und der Prozessverantwortliche aus Gründen des Monitorings und Controllings die laufenden und vergangenen Prozessinstanzen verwalten kann, wird die ebenfalls im Applikations- und Webserver Tomcat eingebettete Camunda Cockpit Webapp genutzt. In einer produktiven Lösung würde diese vermutlich zwar genutzt, würde aber auch die Daten von anderen Process Engines (der Microservices) enthalten, damit alle Daten an einem Ort eingesehen werden können.
2. **Hauptprozess 'Umzug melden'**:
    1. Dies ist der Hauptprozess (End-to-End), welcher aus der Tasklist-Applikation heraus vom Meldepflichtigen gestartet wird.
    2. Da wir auf eine eigene Tasklist-Applikation verzichten und stattdessen die Camunda Webapp Tasklist benutzen, muss der Benutzer bereits an dieser Webapp angemeldet sein, um überhaupt diesen Prozess starten zu können. Dies wäre in einer produktiven Applikation wohl nicht sinnvoll, würde aber aktuell wohl am ehesten dem Modell des Kantons Zürich mit den ZH Services entsprechen, wo man die Steuererklärung auch erst ausfüllen kann, wenn man sich an der Plattform angemeldet hat.
    3. Innerhalb des Hauptprozesses gibt es verschiedene Aktivitäten, die im Folgenden von links nach rechts erläutert werden.
3. **Aufrufprozess 'Umzugsmeldung erfassen und bezahlen'**:
    1. Dies ist derjenige Teil des Hauptprozesses, in welchem der Meldepflichtige über verschiedene Formulare sowie Anbindung von Umsystemen (GWR, Personenregister, usw.) die **Umzugsmeldung erstellt**.
    2. Dieser Teil wurde der Einfachheit halber bewusst **nicht als eigenständiger Microservice ausgelagert**, weil dies sonst dazu führen würde, dass der Benutzer mit zwei Tasklists agieren müsste (inkl. Anmeldung & Co.). In einem produktiven Szenario würde man allerdings ohnehin die Process Engine von der Tasklist-Applikation trennen, so dass dann ein Microservice geeignet wäre.
    3. Streng genommen wäre statt einem über eine **Call Activity** aufgerufenen Prozess ein **Embedded Subprocess** passender. Denn dieser macht für sich alleine ohne Hauptprozess keinen Sinn, kann nun aber aus der Tasklist unsinnigerweise separat gestartet werden. Die nun gewählte Call Activity ist aber tool-bedingt erforderlich, weil der Camunda Modeler (noch) kein Verlinken von Teilprozessmodellen (Embedded Subprocesses) mit dem Hauptprozess erlaubt.
    4. Innerhalb des Aufrufprozesses werden weitere Aufrufprozesse ebenfalls über Call Activities eingebunden aus dem soeben genannten Grund. Was in den jeweiligen Prozessen und Aktivitäten gemacht wird, kann aus den ausführlichen **Element Descriptions** pro BPMN-Element entnommen werden (in Camunda Modeler z.B. im Properties-Panel sichtbar).
4. **'Mit EK-Systemen kommunizieren'**:
    1. In diesem Teil des Hauptprozesses wird die Kommunikation mit den **Einwohner-Kontrollsystemen** der Wegzugs-/Umzugs-/Zuzugsgemeinden erfolgen.
    2. Wie diese aussehen wird, ist später noch zu definieren.
5. **'Erfolgreichen Abschluss mitteilen'**:
    1. In diesem Teil des Hauptprozesses (und auch teilweise in den aufgerufenen Prozessen) erfolgt die automatisierte **Ein-Weg-Kommunikation mit dem Meldepflichtigen**.
    2. Wie diese aussehen wird, ist später noch zu definieren.
6. **'Erfolgreichen Abschluss persistieren'**:
    1. In diesem Teil des Hauptprozesses (und auch teilweise in den aufgerufenen Prozessen) wird die automatisierte **Persistierung von Personendaten und Prozessstatus in der Umzugsplattform-Datenbank** geschehen.
    2. Wie diese aussehen wird, ist später noch zu definieren.

### Personenregister und weitere Umsyteme
Später im Semster von den Studierenden zu erstellen.

## (Technische) Komponenten der Umzugsplattform
Aufgeführt ist der im aktuellen Projekt umgesetzte Stand, nicht die fertige Lösung.

1. **Camunda Spring Boot Starter** 3.0.0 beinhaltend:
    1. Spring Boot-Standardkonfiguration mit Tomcat als Applikations- und Webserver
    2. Camunda Process Engine, REST API und Webapps (Tasklist, Cockpit, Admin) in der Version 7.9.2 (Enterprise Edition), wobei für die Webapps zusätzlich ein eigenes WebJAR-Projekt aus dem lokalen Maven-Repository geladen wird.
    3. Main-Methode in EumzugPlattform2018Application-Klasse
2. **Prozess-Komponenten**:
    1. @EnableProcessApplication-Annotation in EumzugPlattform2018Application-Klasse
    2. processes.xml mit minimaler Konfiguration der Prozessapplikation
    3. Ordner processes mit allen BPMN-Modellen
3. **Persistierungs-Komponenten**:
    1. JDBC-Komponente als Treiber
    2. H2-Datenbank-Unterstützung inklusive Console-Servlet über application.properties-Einstellung
6. **Frontend-Komponenten**:
    1. Die bereits erwähnte Camunda Tasklist App, welche AngularJS, Bootstrap und das Camunda Forms SDK beinhaltet
    3. Ordner static/forms mit einer Embedded Forms-HTML-Datei, welche den HTML-Code für die Forms sowie JavaScript-Code enthalten
7. "Sinnvolle" **Grundkonfiguration** in application.properties für Camunda, Datenbank und Tomcat

## Erforderliche Schritte für das Testen der Applikation
### Voraussetzungen
1. **Camunda Enterprise**: Wenn man die Enterprise Edition von Camunda verwenden will, benötigt man die Zugangsdaten zum Nexus Repository und eine gültige Lizenz. Wie man diese "installiert", steht in den Kommentaren im pom.xml.
4. **Angepasste Camunda Webapps**: Das Projekt https://github.com/zhaw-gpi/be-services-plattform muss gemäss der dort aufgeführten Anleitung konfiguriert und gebuilded sein, damit es im lokalen Maven-Repository zur Verfügung steht.

### Deployment
1. Wenn man die **Enterprise Edition** von Camunda verwenden will, benötigt man die Zugangsdaten zum Nexus Repository und eine gültige Lizenz. Wie man diese "installiert", steht in den Kommentaren im pom.xml.
2. **Erstmalig** oder bei Problemen ein **Clean & Build (Netbeans)**, respektive `mvn clean install` (Cmd) durchführen
3. Bei Änderungen am POM-File oder bei **(Neu)kompilierungsbedarf** genügt ein **Build (Netbeans)**, respektive `mvn install`
4. Für den **Start** ist ein **Run (Netbeans)**, respektive `java -jar .\target\NAME DES JAR-FILES.jar` (Cmd) erforderlich. Dabei wird Tomcat gestartet, die Datenbank erstellt/hochgefahren, Camunda in der Version 7.9 mit den Prozessen und den Eigenschaften (application.properties) hochgefahren.
5. Das **Beenden** geschieht mit **Stop Build/Run (Netbeans)**, respektive **CTRL+C** (Cmd)
6. Falls man die bestehenden **Prozessdaten nicht mehr benötigt** und die Datenbank inzwischen recht angewachsen ist, genügt es, die Datei DATENBANKNAME.mv.db im Wurzelverzeichnis des Projekts zu löschen.

## Nutzung (Testing) der Applikation
### Manuelles Testen
1. In http://localhost:8081 mit dem Benutzer a und Passwort a **anmelden**.
2. **Tasklist**-App starten.
3. Start Process > **Umzug melden** wählen.
4. Dann **durchspielen**, wie dies auch ein Benutzer machen, wobei im Moment erst ein Formular hinterlegt ist und in einer Aktivität gleich zu Beginn gerade so viele Testdaten statisch festgelegt werden, dass der Prozess überhaupt funktionieren kann.
5. **Prozessverlauf überwachen/nachvollziehen**: Um zu sehen, wo der Prozess momentan steckt (Runtime) oder wo abgeschlossene Instanzen durchliefen (History) kann die Cockpit-Webapplikation genutzt werden unter http://localhost:8081.

### Semi-Manuelles Testen
Kommt später

### Automatisiertes Testen
Kommt später

## Prototypische Vereinfachungen
1. **Session-Handling, Benutzerverwaltung, usw.**: In der Realität müssen sich Melde pflichtige nicht anmelden (und damit zunächst registrieren), sondern können eine Meldung auch ohne Anmeldung starten. Unter Verwendung von Camunda Webapps mit User Tasks muss aber ein Benutzer angemeldet sein, damit er den Prozess starten kann und damit ihm Tasks zugewiesen werden können. Wir ignorieren dies und stattdessen ist der Meldepflichtige stets der Benutzer "a", was natürlich dann Probleme gibt, wenn gleichzeitig mehrere Prozessinstanzen laufen, denn dann werden diesem mehrere/verschiedene Aufgaben zugewiesen. Da Name, Vorname, AHV-Nummer, usw. pro Prozessinstanz unterschiedlich ist, sollte es aber dennoch möglich sein, das im Griff zu haben (siehe auch nächster Punkt).
2. **Page Flow vs. BPMN Flow**: In der Realität würde der Meldepflichtige ein Formular nach dem anderen ausfüllen INNERHALB von einem einzigen User Task (oder einem Subprocess), denn verteilt über mehrere User Tasks würde ja bedeuten, dass jedes ausgefüllte Formular einen Eintrag in der Tasklist verursacht und der Benutzer dadurch überhaupt eine Tasklist braucht, was in der Realität nicht zwingend der Fall ist. Auch sollte man in der Realität natürlich im Browser zurück springen können zu einem vorherigen Formular. In der Realität könnte dies z.B. über einen JSF-PageFlow innerhalb des User Tasks gelöst werden. StephenOTT hat [hier](https://forum.camunda.org/t/chaining-user-tasks-to-create-an-interactive-flow-sort-of-a-wizard-style/613/9?u=scepbjoern) gut dargelegt, dass wenn es darum geht, Page Flows (Wizards) abzubilden, ein BPMN-Flow (wie hier umgesetzt) keine geeignete Form ist (allenfalls für das Dokumentieren, aber nicht ausführbar), sondern dann würde man eher eine Lösung wie den form.io-FormBuilder im wizard-Modus nutzen.

## Mitwirkende
Aufgeführt für die fertige Musterlösung, nicht bloss den aktuellen Stand.

1. Björn Scheppler: Hauptarbeit
2. Peter Heinrich: Der stille Support im Hintergrund mit vielen Tipps sowie zuständig
für den Haupt-Stack mit SpringBoot & Co.
3. Raphael Schertenleib: Etliche Formulare
4. Studierende aus dem Herbstsemester 2017:
    1. Gruppe TZb02 (Bekim Kadrija, Jovica Rajic, Luca Belmonte, Simon Bärtschi, Sven 
    Baumann):
        1. Formulare: DatePicker mit Format TT.MM.JJJJ
        2. Mitzuziehende Personen auswählen
        3. Dokumente hochladen
        4. PDF mit allen relevanten Informationen wird als E-Mail-Anhang mitgesendet
        5. Kleinere Verbesserungen an der originalen Musterlösung
    2. Gruppe TZa02 (Alessandro Paradiso, Davor Gavranic, Lars Günthardt, Luka Devcic,
    Robin Chahal):
        1. Wohnungen auswählen
        2. Kleinere Verbesserungen an der originalen Musterlösung
    3. Gruppe TZa03 (Christian Sonntag, Felix Huber, Lukas Brütsch, Tamara Wenk, Vladan 
    Jankovic):
        1. Formulare: Mehrspaltiges Layout
        2. Formulare: Placeholders
        3. Formulare: Bessere Alert- und Help Block-Texte
    4. Gruppe TZb05 (Arbr Wagner, Christoffer Brunner, Leulinda Gutaj, Martin Schwab, 
    Saskia Iwaniw): Formular "Alle Angaben prüfen"