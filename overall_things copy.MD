Aby przenieść aplikację Java (backend) i TypeScript (frontend) do chmury Azure oraz zautomatyzować proces za pomocą Azure DevOps, używając Docker'a, można podążać następującymi krokami:

1. Dockerizacja aplikacji
Najpierw musisz stworzyć pliki Dockerfile dla backendu i frontendu, aby zdokeryzować aplikacje. Poniżej przykłady dla obu:

Backend (Java + Maven) - Dockerfile
Dockerfile
Copy code
# Użyj obrazu z Mavenem do budowania aplikacji
FROM maven:3.8.1-jdk-11 AS build

# Skopiuj pliki projektu
COPY . /app

# Ustaw katalog roboczy
WORKDIR /app

# Zbuduj projekt
RUN mvn clean install -DskipTests

# Drugi etap: Użyj obrazu JDK jako środowisko runtime
FROM openjdk:11-jre-slim

# Skopiuj zbudowany plik JAR z poprzedniego etapu
COPY --from=build /app/target/your-backend.jar /app/your-backend.jar

# Ustaw katalog roboczy
WORKDIR /app

# Wystaw port
EXPOSE 8080

# Uruchom aplikację
ENTRYPOINT ["java", "-jar", "your-backend.jar"]
Frontend (Angular) - Dockerfile
Dockerfile
Copy code
# Użyj obrazu z Node.js do budowania aplikacji
FROM node:14 AS build

# Skopiuj pliki projektu
COPY . /app

# Ustaw katalog roboczy
WORKDIR /app

# Zainstaluj zależności i zbuduj aplikację
RUN npm install
RUN npm run build --prod

# Drugi etap: Użyj serwera Nginx do serwowania aplikacji
FROM nginx:alpine

# Skopiuj zbudowaną aplikację do Nginx
COPY --from=build /app/dist/your-frontend /usr/share/nginx/html

# Wystaw port
EXPOSE 80
2. Utworzenie Azure Container Registry (ACR)
Najpierw utwórz rejestr kontenerów w Azure, aby przechowywać obrazy Docker.

bash
Copy code
az acr create --resource-group your-resource-group --name yourACR --sku Basic
3. Pipeline w Azure DevOps
Następnie zdefiniuj yaml pipeline w Azure DevOps, który zbuduje i wdroży aplikacje. Stwórz plik .yaml w repozytorium, który automatycznie uruchomi procesy CI/CD.

Plik azure-pipelines.yml
yaml
Copy code
trigger:
  branches:
    include:
      - main  # uruchamiaj pipeline na głównej gałęzi

pool:
  vmImage: 'ubuntu-latest'

variables:
  # Zmienne do nazwy rejestru i obrazu
  acrName: 'yourACR.azurecr.io'
  backendImageName: 'your-backend'
  frontendImageName: 'your-frontend'

stages:
  - stage: Build
    jobs:
      - job: Build_Backend
        displayName: 'Build Backend'
        steps:
          - task: Maven@3
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'clean package'
              
          - task: Docker@2
            inputs:
              containerRegistry: '$(acrName)'
              repository: '$(acrName)/$(backendImageName)'
              command: 'buildAndPush'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)

      - job: Build_Frontend
        displayName: 'Build Frontend'
        steps:
          - task: Npm@1
            inputs:
              command: 'install'
              workingDir: 'your-frontend-folder'

          - task: Npm@1
            inputs:
              command: 'run'
              workingDir: 'your-frontend-folder'
              arguments: 'build --prod'

          - task: Docker@2
            inputs:
              containerRegistry: '$(acrName)'
              repository: '$(acrName)/$(frontendImageName)'
              command: 'buildAndPush'
              Dockerfile: 'your-frontend-folder/Dockerfile'
              tags: |
                $(Build.BuildId)

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: Deploy_to_Azure
        displayName: 'Deploy Backend and Frontend to Azure'
        environment: 'Azure'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'your-azure-subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Uruchom backend
                      az container create \
                        --resource-group your-resource-group \
                        --name backend-container \
                        --image $(acrName)/$(backendImageName):$(Build.BuildId) \
                        --cpu 1 --memory 1 \
                        --registry-login-server $(acrName) \
                        --registry-username $(servicePrincipalId) \
                        --registry-password $(servicePrincipalKey) \
                        --dns-name-label your-backend-app \
                        --ports 8080

                      # Uruchom frontend
                      az container create \
                        --resource-group your-resource-group \
                        --name frontend-container \
                        --image $(acrName)/$(frontendImageName):$(Build.BuildId) \
                        --cpu 1 --memory 1 \
                        --registry-login-server $(acrName) \
                        --registry-username $(servicePrincipalId) \
                        --registry-password $(servicePrincipalKey) \
                        --dns-name-label your-frontend-app \
                        --ports 80
4. Skonfiguruj zmienne i uprawnienia
Azure Subscription: W Azure DevOps pipeline użyj "Azure Resource Manager" Service Connection, aby połączyć się z Twoim kontem Azure.
Zmienne: Zdefiniuj acrName, servicePrincipalId, servicePrincipalKey w Azure DevOps jako "secret variables", by umożliwić autoryzację do Azure Container Registry.
5. Zarządzanie Container Instances w Azure
Używając Azure Container Instances (ACI), tworzysz lekkie instancje kontenerów. Skrypty w pipeline powyżej pozwolą Ci uruchomić zarówno backend, jak i frontend jako ACI, gdzie każdy kontener będzie miał własną nazwę DNS i będzie wystawiał odpowiednie porty.

Podsumowanie:
Stwórz pliki Dockerfile dla backendu i frontendu.
Skonfiguruj Azure Container Registry (ACR).
Zdefiniuj pipeline w Azure DevOps, który:
Zbuduje obrazy z backendem i frontendem.
Prześle obrazy do ACR.
Uruchomi instancje kontenerów w Azure Container Instances (ACI).
Skonfiguruj zmienne i połączenia z Azure w Azure DevOps.
To powinno zapewnić CI/CD workflow, który automatycznie zbuduje, zdokeryzuje i wdroży aplikację do Azure przy użyciu Azure DevOps.






You said:
czy backaend i frontend mają działać w ramach tej samej container instance, cyz będą oddzielnie? Jak mają się ze sobą komunikować?
ChatGPT said:
ChatGPT
Backend i frontend powinny działać w osobnych instancjach kontenerów (Azure Container Instances, ACI) z kilku powodów:

Izolacja środowisk: Utrzymując backend i frontend w osobnych kontenerach, zapewniasz lepszą izolację aplikacji. Dzięki temu łatwiej jest skalować i zarządzać poszczególnymi komponentami, a także debugować problemy.

Skalowanie: Backend i frontend mogą mieć różne wymagania dotyczące zasobów (CPU, RAM) i różne obciążenia. W osobnych kontenerach możesz skalować je niezależnie. Na przykład backend może wymagać więcej zasobów podczas przetwarzania danych, podczas gdy frontend głównie serwuje statyczne pliki.

Komunikacja między backendem a frontendem
Aby backend i frontend mogły się ze sobą komunikować, istnieje kilka opcji, które zależą od tego, jak zarządzasz siecią w Azure oraz w jaki sposób aplikacje zostały skonfigurowane.

1. Publiczne punkty końcowe (DNS)
Najprostszym rozwiązaniem jest użycie publicznych nazw DNS dla obu kontenerów, udostępniając backend i frontend z osobnymi publicznymi punktami końcowymi. Frontend może komunikować się z backendem, korzystając z DNS (np. backend-container.westeurope.azurecontainer.io).

W skrócie:

Każdy z kontenerów ma przypisaną własną nazwę DNS (unikalną dla ACI).
Frontend wysyła żądania HTTP do backendu przez jego publiczny DNS, używając np. fetch lub Axios w Angularze.
Przykład:
Backend: backend-container.westeurope.azurecontainer.io:8080
Frontend: frontend-container.westeurope.azurecontainer.io
Frontend wysyła zapytania HTTP do http://backend-container.westeurope.azurecontainer.io:8080/api/endpoint.
W tym przypadku frontend musi być skonfigurowany, aby znał publiczny adres backendu.

2. Wspólna sieć wirtualna (VNet)
Jeśli nie chcesz udostępniać publicznych punktów końcowych lub chcesz ograniczyć dostępność aplikacji tylko do wewnętrznej sieci, możesz skonfigurować wspólną sieć wirtualną (VNet) dla obu kontenerów. W takim przypadku kontenery będą mogły komunikować się bezpośrednio w ramach tej samej sieci prywatnej bez potrzeby wystawiania publicznych adresów IP.

Kroki:

Utwórz sieć wirtualną (VNet) i uruchom oba kontenery w tej samej sieci.
Backend będzie dostępny tylko wewnątrz sieci, a frontend może łączyć się z backendem przez prywatny adres IP lub wewnętrzną nazwę DNS.
3. Azure Kubernetes Service (AKS) (Alternatywa do rozważenia)
Jeśli projekt rozrośnie się i chcesz bardziej zaawansowanej kontroli nad kontenerami, rozważ przeniesienie aplikacji do Azure Kubernetes Service (AKS). Dzięki temu możesz wykorzystać np. Ingress Controller do zarządzania ruchem sieciowym i usługami.

Jak skonfigurować komunikację w aplikacji (Angular + Spring Boot)
1. Frontend (Angular)
W aplikacji frontendowej musisz ustawić endpointy backendu (np. URL serwera API). Jeżeli używasz publicznego punktu końcowego:

typescript
Copy code
const apiUrl = 'http://backend-container.westeurope.azurecontainer.io:8080/api';

this.http.get(`${apiUrl}/endpoint`).subscribe(response => {
  console.log(response);
});
Jeśli backend działa w tej samej sieci VNet (adres prywatny):

typescript
Copy code
const apiUrl = 'http://backend-container/api';
2. Backend (Spring Boot)
Backend działa na porcie 8080 (lub innym), więc jeśli Angular wysyła żądania HTTP do backendu, upewnij się, że w application.properties lub application.yml w Spring Boot skonfigurowano odpowiednie porty i dostępność CORS (jeśli frontend i backend są na różnych domenach).

CORS przykład:

java
Copy code
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://frontend-container.westeurope.azurecontainer.io")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*");
    }
}
Podsumowanie
Backend i frontend działają w osobnych kontenerach dla lepszej izolacji i skalowalności.
Komunikacja między backendem a frontendem może być realizowana przez publiczne DNS'y lub prywatne połączenia w ramach sieci VNet, zależnie od wymagań bezpieczeństwa.
Opcje:
Publiczne DNS'y w ACI — frontend komunikuje się z backendem poprzez publiczny adres.
Sieć VNet — prywatna komunikacja w ramach tej samej sieci wirtualnej (bardziej bezpieczna opcja).
Rozważ AKS w przyszłości, jeśli będziesz potrzebował bardziej zaawansowanej infrastruktury i zarządzania.