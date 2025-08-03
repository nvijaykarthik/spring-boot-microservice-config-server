# code
```
// file-upload.component.ts
import { Component, EventEmitter, Output } from '@angular/core';

@Component({
  selector: 'app-file-upload',
  templateUrl: './file-upload.component.html',
  styleUrls: ['./file-upload.component.css']
})
export class FileUploadComponent {
  @Output() filesSelected = new EventEmitter<File[]>();
  selectedFiles: File[] = [];
  uploadProgress: number[] = [];
  isUploading = false;
  maxFiles = 5;
  errorMessage = '';

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    if (input.files && input.files.length > 0) {
      this.handleFiles(input.files);
    }
  }

  onFileDropped(event: DragEvent): void {
    event.preventDefault();
    const uploadArea = event.currentTarget as HTMLElement;
    uploadArea.classList.remove('drag-over');
    
    if (event.dataTransfer && event.dataTransfer.files.length > 0) {
      this.handleFiles(event.dataTransfer.files);
    }
  }

  handleFiles(fileList: FileList): void {
    this.errorMessage = '';
    
    // Convert FileList to array and check limit
    const newFiles = Array.from(fileList);
    
    if (this.selectedFiles.length + newFiles.length > this.maxFiles) {
      this.errorMessage = `You can upload maximum ${this.maxFiles} files.`;
      return;
    }
    
    // Add new files
    this.selectedFiles = [...this.selectedFiles, ...newFiles.slice(0, this.maxFiles - this.selectedFiles.length)];
    this.filesSelected.emit(this.selectedFiles);
    
    // Initialize progress for each file
    this.uploadProgress = this.selectedFiles.map(() => 0);
    
    // Simulate upload progress
    this.isUploading = true;
    this.simulateUpload();
  }

  simulateUpload(): void {
    const interval = setInterval(() => {
      this.uploadProgress = this.uploadProgress.map(progress => {
        const newProgress = progress + Math.random() * 10;
        return newProgress >= 100 ? 100 : newProgress;
      });
      
      if (this.uploadProgress.every(progress => progress === 100)) {
        clearInterval(interval);
        setTimeout(() => this.isUploading = false, 500);
      }
    }, 200);
  }

  removeFile(index: number): void {
    this.selectedFiles.splice(index, 1);
    this.uploadProgress.splice(index, 1);
    this.filesSelected.emit(this.selectedFiles);
  }

  onDragOver(event: DragEvent): void {
    event.preventDefault();
    const uploadArea = event.currentTarget as HTMLElement;
    uploadArea.classList.add('drag-over');
  }

  onDragLeave(event: DragEvent): void {
    event.preventDefault();
    const uploadArea = event.currentTarget as HTMLElement;
    uploadArea.classList.remove('drag-over');
  }
}
```
# html
```

<!-- file-upload.component.html -->
<div class="upload-container">
  <label class="file-upload-label">
    <input 
      type="file" 
      class="file-input" 
      (change)="onFileSelected($event)"
      accept=".jpg,.jpeg,.png,.pdf,.doc,.docx"
      multiple
    />
    <div class="upload-area"
         (drop)="onFileDropped($event)"
         (dragover)="onDragOver($event)"
         (dragleave)="onDragLeave($event)">
      <div class="upload-icon">
        <svg viewBox="0 0 24 24">
          <path d="M19,13H13V19H11V13H5V11H11V5H13V11H19V13Z" />
        </svg>
      </div>
      <h3 class="upload-title">Drag & Drop files here</h3>
      <p class="upload-subtitle">or click to browse (max {{maxFiles}} files)</p>
    </div>
  </label>
  
  <div class="error-message" *ngIf="errorMessage">
    {{errorMessage}}
  </div>
  
  <div class="files-list" *ngIf="selectedFiles.length > 0">
    <div class="file-item" *ngFor="let file of selectedFiles; let i = index">
      <div class="file-info">
        <span class="file-name">{{file.name}}</span>
        <span class="file-size">{{formatFileSize(file.size)}}</span>
      </div>
      <div class="file-actions">
        <button class="remove-btn" (click)="removeFile(i)">×</button>
      </div>
      <div class="progress-bar" *ngIf="isUploading">
        <div class="progress" [style.width.%]="uploadProgress[i]"></div>
      </div>
    </div>
  </div>
  
  <div class="upload-status" *ngIf="!isUploading && uploadProgress.length > 0 && uploadProgress.every(p => p === 100)">
    <p class="success-message">Upload completed successfully!</p>
  </div>
</div>
```
# css
```
/* file-upload.component.css */
.upload-container {
  max-width: 500px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.file-upload-label {
  display: block;
  cursor: pointer;
}

.file-input {
  display: none;
}

.upload-area {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 40px 20px;
  text-align: center;
  transition: all 0.3s ease;
  background-color: #f9f9f9;
}

.upload-area:hover {
  border-color: #4a89dc;
  background-color: #f0f7ff;
}

.upload-icon {
  width: 60px;
  height: 60px;
  margin: 0 auto 15px;
  fill: #4a89dc;
}

.upload-title {
  margin: 0 0 10px;
  color: #333;
  font-size: 18px;
  font-weight: 600;
}

.upload-subtitle {
  margin: 0;
  color: #777;
  font-size: 14px;
}

/* Drag and drop styling */
.upload-area.drag-over {
  border-color: #4a89dc;
  background-color: #e6f0ff;
}

/* File info styling */
.file-info {
  margin-top: 20px;
  padding: 15px;
  border-radius: 6px;
  background-color: #f5f5f5;
}

.file-info p {
  margin: 5px 0;
  color: #555;
}

.progress-bar {
  height: 6px;
  background-color: #e0e0e0;
  border-radius: 3px;
  margin-top: 10px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background-color: #4a89dc;
  width: 0%;
  transition: width 0.3s ease;
}

/* Responsive adjustments */
@media (max-width: 600px) {
  .upload-container {
    padding: 10px;
  }
  
  .upload-area {
    padding: 30px 15px;
  }
}
.success-message {
  color: #4caf50;
  font-weight: 500;
  margin-top: 10px;
}

/* Animation for upload complete */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.file-info {
  animation: fadeIn 0.5s ease;
}
/* Add to your existing CSS */
.error-message {
  color: #f44336;
  margin: 10px 0;
  padding: 10px;
  background-color: #ffebee;
  border-radius: 4px;
}

.files-list {
  margin-top: 20px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
}

.file-item {
  padding: 12px 15px;
  border-bottom: 1px solid #eee;
}

.file-item:last-child {
  border-bottom: none;
}

.file-info {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.file-name {
  font-weight: 500;
  color: #333;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

.file-size {
  color: #777;
  font-size: 0.9em;
  margin-left: 10px;
}

.file-actions {
  text-align: right;
}

.remove-btn {
  background: none;
  border: none;
  color: #f44336;
  font-size: 1.2em;
  cursor: pointer;
  padding: 0 5px;
}

.remove-btn:hover {
  color: #d32f2f;
}

.upload-status {
  margin-top: 15px;
  text-align: center;
}
```
```
formatFileSize(bytes: number): string {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}
```


```
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <app-file-upload (filesSelected)="handleFiles($event)"></app-file-upload>
  `
})
export class AppComponent {
  handleFiles(files: File[]): void {
    console.log('Files selected:', files);
    // Here you would typically upload the files to your server
  }
}
```
