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