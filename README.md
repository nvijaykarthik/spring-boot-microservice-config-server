CREATE TABLE IF NOT EXISTS PROPERTIES (
  APPLICATION VARCHAR(255) NOT NULL,
  PROFILE VARCHAR(255) NOT NULL,
  LABEL VARCHAR(255) NOT NULL,
  KEY VARCHAR(255) NOT NULL,
  VALUE VARCHAR(4096),
  ACTIVE BOOLEAN DEFAULT TRUE,
  LAST_UPDATED TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (APPLICATION, PROFILE, LABEL, KEY)
);

CREATE INDEX IDX_PROPERTIES_ACTIVE ON PROPERTIES(ACTIVE);

********
INSERT INTO PROPERTIES (APPLICATION, PROFILE, LABEL, KEY, VALUE, ACTIVE)
VALUES 
('my-client-app', 'dev', 'main', 'active.property', 'enabled-value', TRUE),
('my-client-app', 'dev', 'main', 'feature.flag', 'true', TRUE);

-- Inactive properties
INSERT INTO PROPERTIES (APPLICATION, PROFILE, LABEL, KEY, VALUE, ACTIVE)
VALUES 
('my-client-app', 'dev', 'main', 'old.property', 'deprecated-value', FALSE),
('my-client-app', 'dev', 'main', 'legacy.flag', 'false', FALSE);

spring:
  cloud:
    config:
      server:
        jdbc:
          sql: SELECT KEY, VALUE from PROPERTIES where APPLICATION=? and PROFILE=? and LABEL=? and ACTIVE=TRUE

  import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.util.StringUtils;

@RestController
@RequestMapping("/api/config")
public class ConfigFileUploadController {

    // ... existing fields and constructor
    
    @PostMapping("/upload")
    public ResponseEntity<String> uploadConfigFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam("application") String application,
            @RequestParam("profile") String profile,
            @RequestParam("label") String label) {
        
        try {
            Map<String, String> properties;
            
            if (isYamlFile(file)) {
                properties = parseWithSpringYaml(file);
            } else {
                properties = parsePropertiesFile(file);
            }
            
            savePropertiesToDatabase(application, profile, label, properties);
            return ResponseEntity.ok("Uploaded " + properties.size() + " properties");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error: " + e.getMessage());
        }
    }
    
    private boolean isYamlFile(MultipartFile file) {
        String filename = StringUtils.getFilenameExtension(file.getOriginalFilename());
        return "yml".equalsIgnoreCase(filename) || "yaml".equalsIgnoreCase(filename);
    }
    
    private Map<String, String> parseWithSpringYaml(MultipartFile file) throws IOException {
        YamlPropertiesFactoryBean yamlFactory = new YamlPropertiesFactoryBean();
        yamlFactory.setResources(new ByteArrayResource(file.getBytes()));
        Properties props = yamlFactory.getObject();
        
        Map<String, String> result = new HashMap<>();
        for (String name : props.stringPropertyNames()) {
            result.put(name, props.getProperty(name));
        }
        return result;
    }
    
    private Map<String, String> parsePropertiesFile(MultipartFile file) throws IOException {
        Properties props = new Properties();
        props.load(file.getInputStream());
        
        Map<String, String> result = new HashMap<>();
        for (String name : props.stringPropertyNames()) {
            result.put(name, props.getProperty(name));
        }
        return result;
    }
    
    // ... rest of the existing methods
}



private void savePropertiesToDatabase(String application, String profile, String label, 
                                    Map<String, String> properties) {
    
    // First deactivate all existing properties for this app/profile/label
    propertyRepository.deactivateAllProperties(application, profile, label);
    
    // Insert or update each property
    for (Map.Entry<String, String> entry : properties.entrySet()) {
        Optional<Property> existing = propertyRepository.findById(
            new PropertyId(application, profile, label, entry.getKey()));
        
        if (existing.isPresent()) {
            // Update existing property
            Property prop = existing.get();
            prop.setValue(entry.getValue());
            prop.setActive(true);
            prop.setLastUpdated(new Timestamp(System.currentTimeMillis()));
            propertyRepository.save(prop);
        } else {
            // Insert new property
            Property prop = new Property();
            prop.setApplication(application);
            prop.setProfile(profile);
            prop.setLabel(label);
            prop.setKey(entry.getKey());
            prop.setValue(entry.getValue());
            prop.setActive(true);
            prop.setLastUpdated(new Timestamp(System.currentTimeMillis()));
            propertyRepository.save(prop);
        }
    }
}


@PostMapping("/upload-multiple")
public ResponseEntity<String> uploadMultipleConfigFiles(
        @RequestParam("files") MultipartFile[] files,
        @RequestParam("application") String application,
        @RequestParam("profile") String profile,
        @RequestParam("label") String label) {
    
    if (files == null || files.length == 0) {
        return ResponseEntity.badRequest().body("No files provided");
    }

    try {
        Map<String, String> allProperties = new HashMap<>();
        
        for (MultipartFile file : files) {
            Map<String, String> fileProperties;
            
            if (isYamlFile(file)) {
                fileProperties = parseWithSpringYaml(file);
            } else {
                fileProperties = parsePropertiesFile(file);
            }
            
            // Merge properties, with later files overriding earlier ones
            allProperties.putAll(fileProperties);
        }
        
        savePropertiesToDatabase(application, profile, label, allProperties);
        return ResponseEntity.ok("Uploaded " + allProperties.size() + 
                              " properties from " + files.length + " files");
    } catch (Exception e) {
        return ResponseEntity.badRequest().body("Error processing files: " + e.getMessage());
    }
}


@PostMapping("/upload-ordered")
public ResponseEntity<String> uploadOrderedFiles(
        @RequestPart("files") List<MultipartFile> files,
        @RequestParam("application") String application,
        @RequestParam("profile") String profile,
        @RequestParam("label") String label) {
    
    // Process files in reverse order so first files have lowest priority
    Collections.reverse(files);
    
    Map<String, String> combinedProperties = new LinkedHashMap<>();
    
    for (MultipartFile file : files) {
        Map<String, String> currentProps;
        try {
            currentProps = isYamlFile(file) ? 
                parseWithSpringYaml(file) : 
                parsePropertiesFile(file);
        } catch (IOException e) {
            return ResponseEntity.badRequest()
                .body("Error processing " + file.getOriginalFilename() + ": " + e.getMessage());
        }
        
        // Later files (now earlier in the reversed list) will override existing properties
        combinedProperties.putAll(currentProps);
    }
    
    savePropertiesToDatabase(application, profile, label, combinedProperties);
    return ResponseEntity.ok("Processed " + files.size() + " files");
}


private void validateFiles(MultipartFile[] files) throws IllegalArgumentException {
    if (files == null || files.length == 0) {
        throw new IllegalArgumentException("At least one file must be provided");
    }
    
    for (MultipartFile file : files) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("File " + file.getOriginalFilename() + " is empty");
        }
        
        String ext = StringUtils.getFilenameExtension(file.getOriginalFilename());
        if (!"yml".equalsIgnoreCase(ext) && 
            !"yaml".equalsIgnoreCase(ext) && 
            !"properties".equalsIgnoreCase(ext)) {
            throw new IllegalArgumentException(
                "Unsupported file type: " + file.getOriginalFilename());
        }
    }
}


MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
body.add("application", "myapp");
body.add("profile", "dev");
body.add("label", "main");

body.add("files", new FileSystemResource("base.yml"));
body.add("files", new FileSystemResource("secrets.properties"));

HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.MULTIPART_FORM_DATA);

ResponseEntity<String> response = restTemplate.exchange(
    "http://config-server:8888/api/config/upload-multiple",
    HttpMethod.POST,
    new HttpEntity<>(body, headers),
    String.class);



-- Temporary staging table
CREATE TABLE IF NOT EXISTS PROPERTIES_STAGING (
  ID BIGINT AUTO_INCREMENT PRIMARY KEY,
  APPLICATION VARCHAR(255) NOT NULL,
  PROFILE VARCHAR(255) NOT NULL,
  LABEL VARCHAR(255) NOT NULL,
  KEY VARCHAR(255) NOT NULL,
  VALUE VARCHAR(4096),
  ACTION VARCHAR(20) NOT NULL, -- 'ADD', 'UPDATE', 'DELETE'
  CURRENT_VALUE VARCHAR(4096), -- Only for updates/deletes
  STATUS VARCHAR(20) DEFAULT 'PENDING', -- 'PENDING', 'APPROVED', 'REJECTED'
  CREATED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CREATED_BY VARCHAR(255),
  UNIQUE KEY (APPLICATION, PROFILE, LABEL, KEY)
);

-- Add versioning to main table
ALTER TABLE PROPERTIES ADD COLUMN VERSION INT DEFAULT 1;

@Service
public class ConfigStagingService {
    
    @Autowired
    private PropertyRepository propertyRepository;
    
    @Autowired
    private PropertyStagingRepository stagingRepository;
    
    @Transactional
    public void stageChanges(String application, String profile, String label, 
                           Map<String, String> newProperties, String username) {
        
        // Get current active properties
        Map<String, String> currentProperties = propertyRepository
            .findActiveProperties(application, profile, label)
            .stream()
            .collect(Collectors.toMap(Property::getKey, Property::getValue));
        
        // Process changes
        newProperties.forEach((key, newValue) -> {
            PropertyStaging staging = new PropertyStaging();
            staging.setApplication(application);
            staging.setProfile(profile);
            staging.setLabel(label);
            staging.setKey(key);
            staging.setValue(newValue);
            staging.setCreatedBy(username);
            
            if (currentProperties.containsKey(key)) {
                String currentValue = currentProperties.get(key);
                if (!currentValue.equals(newValue)) {
                    staging.setAction("UPDATE");
                    staging.setCurrentValue(currentValue);
                    stagingRepository.save(staging);
                }
            } else {
                staging.setAction("ADD");
                stagingRepository.save(staging);
            }
        });
        
        // Identify deletions
        currentProperties.keySet().stream()
            .filter(key -> !newProperties.containsKey(key))
            .forEach(key -> {
                PropertyStaging staging = new PropertyStaging();
                staging.setApplication(application);
                staging.setProfile(profile);
                staging.setLabel(label);
                staging.setKey(key);
                staging.setAction("DELETE");
                staging.setCurrentValue(currentProperties.get(key));
                staging.setCreatedBy(username);
                stagingRepository.save(staging);
            });
    }
    
    public StagingDiff getStagingDiff(String application, String profile, String label) {
        List<PropertyStaging> changes = stagingRepository
            .findPendingChanges(application, profile, label);
        
        return new StagingDiff(
            changes.stream()
                .filter(c -> "ADD".equals(c.getAction()))
                .collect(Collectors.toList()),
            changes.stream()
                .filter(c -> "UPDATE".equals(c.getAction()))
                .collect(Collectors.toList()),
            changes.stream()
                .filter(c -> "DELETE".equals(c.getAction()))
                .collect(Collectors.toList())
        );
    }
    
    @Transactional
    public void approveChanges(String application, String profile, String label, String username) {
        List<PropertyStaging> changes = stagingRepository
            .findPendingChanges(application, profile, label);
        
        // Process adds and updates
        changes.stream()
            .filter(c -> !"DELETE".equals(c.getAction()))
            .forEach(change -> {
                Optional<Property> existing = propertyRepository.findById(
                    new PropertyId(application, profile, label, change.getKey()));
                
                if (existing.isPresent()) {
                    Property prop = existing.get();
                    prop.setValue(change.getValue());
                    prop.setActive(true);
                    prop.setVersion(prop.getVersion() + 1);
                    propertyRepository.save(prop);
                } else {
                    Property prop = new Property();
                    prop.setApplication(application);
                    prop.setProfile(profile);
                    prop.setLabel(label);
                    prop.setKey(change.getKey());
                    prop.setValue(change.getValue());
                    prop.setActive(true);
                    prop.setVersion(1);
                    propertyRepository.save(prop);
                }
            });
        
        // Process deletes
        changes.stream()
            .filter(c -> "DELETE".equals(c.getAction()))
            .forEach(change -> {
                propertyRepository.findById(
                    new PropertyId(application, profile, label, change.getKey()))
                    .ifPresent(prop -> {
                        prop.setActive(false);
                        propertyRepository.save(prop);
                    });
            });
        
        // Mark staging records as approved
        stagingRepository.updateStatusForChanges(
            application, profile, label, "APPROVED", username);
    }
    
    @Transactional
    public void rejectChanges(String application, String profile, String label, String username) {
        stagingRepository.updateStatusForChanges(
            application, profile, label, "REJECTED", username);
    }
}


public class StagingDiff {
    private List<PropertyStaging> additions;
    private List<PropertyStaging> updates;
    private List<PropertyStaging> deletions;
    
    // Constructor, getters, setters
}

public class PropertyStagingDTO {
    private String application;
    private String profile;
    private String label;
    private String key;
    private String newValue;
    private String currentValue; // Only for updates/deletes
    private String action;
    
    // Getters, setters
}


@RestController
@RequestMapping("/api/config/staging")
public class ConfigStagingController {
    
    @Autowired
    private ConfigStagingService stagingService;
    
    @PostMapping("/upload")
    public ResponseEntity<String> stageConfigurationChanges(
            @RequestParam("files") MultipartFile[] files,
            @RequestParam("application") String application,
            @RequestParam("profile") String profile,
            @RequestParam("label") String label,
            @RequestHeader("X-Username") String username) {
        
        try {
            Map<String, String> allProperties = new HashMap<>();
            
            for (MultipartFile file : files) {
                Map<String, String> fileProperties = isYamlFile(file) ?
                    parseWithSpringYaml(file) :
                    parsePropertiesFile(file);
                allProperties.putAll(fileProperties);
            }
            
            stagingService.stageChanges(application, profile, label, allProperties, username);
            return ResponseEntity.ok("Changes staged for review");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error staging changes: " + e.getMessage());
        }
    }
    
    @GetMapping("/diff")
    public ResponseEntity<StagingDiff> getStagingDiff(
            @RequestParam("application") String application,
            @RequestParam("profile") String profile,
            @RequestParam("label") String label) {
        
        return ResponseEntity.ok(stagingService.getStagingDiff(application, profile, label));
    }
    
    @PostMapping("/approve")
    public ResponseEntity<String> approveChanges(
            @RequestParam("application") String application,
            @RequestParam("profile") String profile,
            @RequestParam("label") String label,
            @RequestHeader("X-Username") String username) {
        
        stagingService.approveChanges(application, profile, label, username);
        return ResponseEntity.ok("Changes approved and applied");
    }
    
    @PostMapping("/reject")
    public ResponseEntity<String> rejectChanges(
            @RequestParam("application") String application,
            @RequestParam("profile") String profile,
            @RequestParam("label") String label,
            @RequestHeader("X-Username") String username) {
        
        stagingService.rejectChanges(application, profile, label, username);
        return ResponseEntity.ok("Changes rejected");
    }
}


public interface PropertyStagingRepository extends JpaRepository<PropertyStaging, Long> {
    
    @Query("SELECT s FROM PropertyStaging s " +
           "WHERE s.application = :application " +
           "AND s.profile = :profile " +
           "AND s.label = :label " +
           "AND s.status = 'PENDING'")
    List<PropertyStaging> findPendingChanges(
        @Param("application") String application,
        @Param("profile") String profile,
        @Param("label") String label);
    
    @Modifying
    @Query("UPDATE PropertyStaging s SET s.status = :status, s.approvedBy = :username " +
           "WHERE s.application = :application " +
           "AND s.profile = :profile " +
           "AND s.label = :label " +
           "AND s.status = 'PENDING'")
    void updateStatusForChanges(
        @Param("application") String application,
        @Param("profile") String profile,
        @Param("label") String label,
        @Param("status") String status,
        @Param("username") String username);
}
