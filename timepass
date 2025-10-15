import React, { useEffect, useState, useRef } from "react";
import "./filemanagement.css";
import { FaDownload, FaCheckCircle, FaTimesCircle, FaFilter } from "react-icons/fa";
import DataService from "../../Services/DataService";
import { toast } from "react-toastify";
import JSZip from "jszip";
import Loader from "../../Components/Loader/Loader";
import { FaInfo } from "react-icons/fa";

const ACCEPTED_EXT = [".csv", ".xlsx", ".zip"];
const MAX_FILES = 5;
const MAX_TOTAL_SIZE = 50 * 1024 * 1024; // 50 MB
const MAX_SINGLE_FILE = 10 * 1024 * 1024; // 10 MB
const ITEMS_PER_PAGE = 5; // Changed to 5 files per page
function prettySize(bytes) {
    if (bytes >= 1024 * 1024) return (bytes / (1024 * 1024)).toFixed(2) + " MB";
    return (bytes / 1024).toFixed(2) + " KB";
}
export default function FileManagement({ userId, userName, userFullName, subscriptionId }) {
    const [files, setFiles] = useState([]);
    const [filesMeta, setFilesMeta] = useState([]);
    const [selectedFiles, setSelectedFiles] = useState([]);
    const [loading, setLoading] = useState(false);
    const [uploading, setUploading] = useState(false);
    const [uploadType, setUploadType] = useState("daily");
    const [downloading, setDownloading] = useState(false);
    const [currentPage, setCurrentPage] = useState(1);
    const [filters, setFilters] = useState({
        uploadType: "",
        uploadDate: "",
        status: "",
        validation: "",
    });
    const [showFilterDropdown, setShowFilterDropdown] = useState(null); // Track which dropdown is open
    const dailyInputRef = useRef(null);
    const monthlyInputRef = useRef(null);
    useEffect(() => {
        if (userId) fetchFileRecords();
        window.refreshFileManagement = fetchFileRecords;
        return () => { window.refreshFileManagement = null; };
    }, [userId]);
    async function fetchFileRecords() {
        try {
            setLoading(true);
            const res = await DataService.getFileUploadRecords(userId);
            if (Array.isArray(res) && res.length > 0) {
                const formatted = res.map((item) => ({
                    id: item.id,
                    fileName: item.file_name,
                    uploadType:
                        item.upload_type?.charAt(0).toUpperCase() +
                        item.upload_type?.slice(1) || "Unknown",
                    uploadDate: new Date(item.upload_date),
                    uploadDateString: new Date(item.upload_date).toLocaleDateString("en-US", {
                        month: "short",
                        day: "numeric",
                        hour: "2-digit",
                        minute: "2-digit",
                    }),
                    status: item.status,
                    validation: item.status === "Uploaded"
                        ? "Validated"
                        : item.status === "Failed"
                            ? "Failed"
                            : "Pending",
                    errorMessage: item.error_message || "",
                    size: item.size || 0,
                }));
                setFiles(formatted);
            } else {
                setFiles([]);
                toast.info("No uploaded files found.");
            }
        } catch (err) {
            console.error("Error fetching uploaded files:", err);
            toast.error("Failed to fetch file records");
        } finally {
            setLoading(false);
        }
    }
    // Filter and sort files
    const filteredFiles = files
        .filter((file) => {
            return (
                file.uploadType.toLowerCase().includes(filters.uploadType.toLowerCase()) &&
                file.status.toLowerCase().includes(filters.status.toLowerCase()) &&
                file.validation.toLowerCase().includes(filters.validation.toLowerCase())
            );
        })
        .sort((a, b) => {
            if (filters.uploadDate === "Latest") {
                return b.uploadDate - a.uploadDate;
            } else if (filters.uploadDate === "Oldest") {
                return a.uploadDate - b.uploadDate;
            }
            return 0; // No sorting if filter is empty
        });
    // Pagination logic
    const totalPages = Math.ceil(filteredFiles.length / ITEMS_PER_PAGE);
    const paginatedFiles = filteredFiles.slice(
        (currentPage - 1) * ITEMS_PER_PAGE,
        currentPage * ITEMS_PER_PAGE
    );
    // Handle page change
    const handlePageChange = (page) => {
        if (page >= 1 && page <= totalPages) {
            setCurrentPage(page);
        }
    };
    // Handle filter change
    const handleFilterChange = (field, value) => {
        setFilters((prev) => ({ ...prev, [field]: value }));
        setCurrentPage(1);
        setShowFilterDropdown(null); // Close dropdown after selection
    };
    // Toggle filter dropdown
    const toggleFilterDropdown = (field) => {
        setShowFilterDropdown(showFilterDropdown === field ? null : field);
    };
    function fileExt(name) {
        const idx = name.lastIndexOf(".");
        return idx === -1 ? "" : name.slice(idx).toLowerCase();
    }
    async function validateAndAddFiles(listOfFiles, target) {
        const incoming = Array.from(listOfFiles);
        if (selectedFiles.length + incoming.length > MAX_FILES) {
            toast.warning(`You can only upload up to ${MAX_FILES} files.`);
            return;
        }
        const validFiles = [];
        for (const file of incoming) {
            const ext = fileExt(file.name);
            if (!ACCEPTED_EXT.includes(ext)) {
                toast.warning(`File format not supported: ${file.name}`);
                continue;
            }
            if (file.size > MAX_SINGLE_FILE) {
                toast.warning(`${file.name} is larger than ${prettySize(MAX_SINGLE_FILE)}.`);
                continue;
            }
            if (ext === ".zip") {
                const zip = new JSZip();
                try {
                    const zipContent = await zip.loadAsync(file);
                    const entries = Object.values(zipContent.files).filter((f) => !f.dir);
                    if (entries.length > 5) {
                        toast.warning(`${file.name} contains more than 5 files.`);
                        continue;
                    }
                    const oversizedFiles = [];
                    for (const entry of entries) {
                        const content = await entry.async("uint8array");
                        if (content.byteLength > MAX_SINGLE_FILE) oversizedFiles.push(entry.name);
                    }
                    if (oversizedFiles.length > 0) {
                        toast.warning(`Zip file "${file.name}" has files larger than 10 MB: ${oversizedFiles.join(", ")}`);
                        continue;
                    }
                    validFiles.push(file);
                } catch (err) {
                    toast.error(`Failed to read zip file: ${file.name}`);
                    continue;
                }
            } else validFiles.push(file);
        }
        const curTotal = selectedFiles.reduce((s, f) => s + f.size, 0);
        const incomingTotal = validFiles.reduce((s, f) => s + f.size, 0);
        if (curTotal + incomingTotal > MAX_TOTAL_SIZE) {
            toast.warning("Total size of files cannot exceed 50 MB.");
            return;
        }
        const newSelected = [...selectedFiles, ...validFiles];
        const newMeta = [
            ...filesMeta,
            ...validFiles.map((f) => ({
                FileName: f.name,
                Size: prettySize(f.size),
                Status: "Pending",
                ValidationDetails: "",
                uploadType: target,
            })),
        ];
        setSelectedFiles(newSelected);
        setFilesMeta(newMeta);
        setUploadType(target);
    }
    function onDragOver(e) { e.preventDefault(); }
    function onDrop(e, target) { e.preventDefault(); validateAndAddFiles(e.dataTransfer.files, target); }
    function onInputFiles(e, target) { validateAndAddFiles(e.target.files, target); e.target.value = ""; }
    function removeFile(index) {
        const newSel = [...selectedFiles];
        const newMeta = [...filesMeta];
        newSel.splice(index, 1);
        newMeta.splice(index, 1);
        setSelectedFiles(newSel);
        setFilesMeta(newMeta);
    }
    async function onValidate() {
        setFilesMeta((m) => m.map((f) => ({ ...f, Status: "Validated" })));
        toast.success("Files validated successfully");
    }
    async function onSubmit() {
        if (selectedFiles.length === 0) { toast.warning("No files uploaded"); return; }
        setUploading(true);
        try {
            const form = new FormData();
            selectedFiles.forEach((f) => form.append("files", f, f.name));
            form.append("userId", userId);
            form.append("subscriptionId", subscriptionId);
            form.append("uploadType", uploadType);
            form.append("username", userName);
            await DataService.uploadFiles(form);
            const fileDetailsPayload = {
                userName,
                userId,
                uploadType,
                files: filesMeta.map((f) => ({ FileName: f.FileName })),
            };
            await DataService.storeFileUploadDetails(fileDetailsPayload);
            setFilesMeta((m) => m.map((f) => ({ ...f, Status: "Uploaded", ValidationDetails: "" })));
            toast.success("Files uploaded successfully");
            setSelectedFiles([]);
            setFilesMeta([]);
            fetchFileRecords();
        } catch (err) {
            console.error("Error uploading files:", err);
            setFilesMeta((m) => m.map((f) => ({ ...f, Status: "Error", ValidationDetails: err.message || "Upload error" })));
            toast.error("Error uploading files");
        } finally { setUploading(false); }
    }
    function onViewErrors(file) {
        if (file.validation === "Failed" && file.errorMessage) {
            toast.error(file.errorMessage, { autoClose: 5000 });
        } else {
            toast.info("No Error Found", { autoClose: 3000 });
        }
    }
    async function onDownloadFile(fileName) {
        setDownloading("Downloading file...");
        try {
            const res = await DataService.downloadFile(userId, fileName);
            if (!res?.url) throw new Error("File URL not found");
            const response = await fetch(res.url);
            const blob = await response.blob();
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement("a");
            a.href = url;
            a.download = fileName;
            document.body.appendChild(a);
            a.click();
            a.remove();
            window.URL.revokeObjectURL(url);
        } catch (err) {
            console.error("Download failed:", err);
            toast.error("Failed to download file");
        } finally { setDownloading(false); }
    }
    return (
        <div className="file-mgmt-container">
            {downloading && <Loader message={downloading} />}
            {uploading && <Loader message="Uploading files..." />}
            <div className="file-mgmt-header">
                <div>
                    <h3>File Management</h3>
                    <p className="muted">Upload and validate files</p>
                </div>
                <div className="upload-buttons">
                    <button className="btn-upload-daily" onClick={() => setUploadType("daily") || dailyInputRef.current?.click()}>
                        Upload Daily
                    </button>
                    <button className="btn-upload-monthly" onClick={() => setUploadType("monthly") || monthlyInputRef.current?.click()}>
                        Upload Monthly
                    </button>
                </div>
            </div>
            <input ref={dailyInputRef} type="file" multiple style={{ display: "none" }} onChange={(e) => onInputFiles(e, "daily")} accept=".csv,.xlsx,.zip" />
            <input ref={monthlyInputRef} type="file" multiple style={{ display: "none" }} onChange={(e) => onInputFiles(e, "monthly")} accept=".csv,.xlsx,.zip" />
            {/* Selected files table */}
            <div className="file-actions">
                <div className="file-list">
                    {filesMeta.length === 0 ? (
                        <p className="muted">No files selected.</p>
                    ) : (
                        <table className="mini-table">
                            <thead>
                                <tr>
                                    <th>#</th>
                                    <th>File Name</th>
                                    <th>Size</th>
                                    <th>Upload Type</th>
                                    <th>Status</th>
                                    <th></th>
                                </tr>
                            </thead>
                            <tbody>
                                {filesMeta.map((m, i) => (
                                    <tr key={i}>
                                        <td>{i + 1}</td>
                                        <td>{m.FileName}</td>
                                        <td>{m.Size}</td>
                                        <td>{m.uploadType}</td>
                                        <td>{m.Status}</td>
                                        <td>
                                            <button className="link-small" onClick={() => removeFile(i)}>Remove</button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    )}
                </div>
                <div className="action-buttons">
                    <button onClick={onValidate} disabled={filesMeta.length === 0}>Validate</button>
                    <button onClick={onSubmit} disabled={filesMeta.length === 0 || uploading}>
                        {uploading ? "Uploading..." : "Submit"}
                    </button>
                </div>
            </div>
            {/* Existing file records table with filters */}
            {loading ? (
                <p className="muted">Loading file records...</p>
            ) : filteredFiles.length === 0 ? (
                <p className="muted">No files found for this user.</p>
            ) : (
                <>
                    <table className="file-table">
                        <thead>
                            <tr>
                                <th>Filename</th>
                                <th>
                                    Type
                                    <FaFilter
                                        className="filter-icon"
                                        onClick={() => toggleFilterDropdown("uploadType")}
                                    />
                                    {showFilterDropdown === "uploadType" && (
                                        <div className="filter-dropdown">
                                            <select
                                                value={filters.uploadType}
                                                onChange={(e) => handleFilterChange("uploadType", e.target.value)}
                                            >
                                                <option value="">All</option>
                                                <option value="Daily">Daily</option>
                                                <option value="Monthly">Monthly</option>
                                            </select>
                                        </div>
                                    )}
                                </th>
                                <th>
                                    Upload Date
                                    <FaFilter
                                        className="filter-icon"
                                        onClick={() => toggleFilterDropdown("uploadDate")}
                                    />
                                    {showFilterDropdown === "uploadDate" && (
                                        <div className="filter-dropdown">
                                            <select
                                                value={filters.uploadDate}
                                                onChange={(e) => handleFilterChange("uploadDate", e.target.value)}
                                            >
                                                <option value="">All</option>
                                                <option value="Latest">Latest</option>
                                                <option value="Oldest">Oldest</option>
                                            </select>
                                        </div>
                                    )}
                                </th>
                                <th>
                                    Status
                                    <FaFilter
                                        className="filter-icon"
                                        onClick={() => toggleFilterDropdown("status")}
                                    />
                                    {showFilterDropdown === "status" && (
                                        <div className="filter-dropdown">
                                            <select
                                                value={filters.status}
                                                onChange={(e) => handleFilterChange("status", e.target.value)}
                                            >
                                                <option value="">All</option>
                                                <option value="Uploaded">Uploaded</option>
                                                <option value="Failed">Failed</option>
                                                <option value="Pending">Pending</option>
                                            </select>
                                        </div>
                                    )}
                                </th>
                                <th>
                                    Validation
                                    <FaFilter
                                        className="filter-icon"
                                        onClick={() => toggleFilterDropdown("validation")}
                                    />
                                    {showFilterDropdown === "validation" && (
                                        <div className="filter-dropdown">
                                            <select
                                                value={filters.validation}
                                                onChange={(e) => handleFilterChange("validation", e.target.value)}
                                            >
                                                <option value="">All</option>
                                                <option value="Validated">Validated</option>
                                                <option value="Failed">Failed</option>
                                                <option value="Pending">Pending</option>
                                            </select>
                                        </div>
                                    )}
                                </th>
                                {/* <th>Template</th> */}
                                <th>Validate</th>
                                <th>Action</th>
                            </tr>
                        </thead>
                        <tbody>
                            {paginatedFiles.map((file, index) => (
                                <tr key={index}>
                                    <td>
                                        <span className="file-link">{file.fileName}</span>
                                        {file.validation === "Failed" && (
                                            <div className="error-text">Missing mandatory fields: Employee ID</div>
                                        )}
                                    </td>
                                    <td>{file.uploadType}</td>
                                    <td>{file.uploadDateString}</td>
                                    <td>
                                        <span
                                            className={`status-badge ${file.status === "Uploaded"
                                                ? "status-ready"
                                                : file.status === "Failed"
                                                    ? "status-error"
                                                    : "status-uploaded"
                                                }`}
                                        >
                                            {file.status}
                                        </span>
                                    </td>
                                    <td>
                                        {file.validation === "Validated" ? (
                                            <span className="validation-badge success">
                                                <FaCheckCircle /> Validated
                                            </span>
                                        ) : file.validation === "Failed" ? (
                                            <span className="validation-badge failed">
                                                <FaTimesCircle /> Failed
                                            </span>
                                        ) : (
                                            <span className="validation-badge pending">Pending</span>
                                        )}
                                    </td>
                                    {/* <td>
                                        <span className="template-link"><FaDownload /> CSV</span>
                                    </td> */}
                                    <td>
                                        {file.validation === "Validated" ? (
                                            <FaCheckCircle className="validate-icon success" />
                                        ) : (
                                            <button className="validate-btn">Validate</button>
                                        )}
                                    </td>
                                    <td className="action-column">
                                        <span
                                            className="action-icon"
                                            title="View Errors"
                                            onClick={() => onViewErrors(file)}
                                        >
                                            <FaInfo />
                                        </span>
                                    </td>
                                </tr>
                            ))}
                        </tbody>
                    </table>
                    {/* Pagination controls */}
                    <div className="pagination">
                        <button
                            onClick={() => handlePageChange(currentPage - 1)}
                            disabled={currentPage === 1}
                        >
                            Previous
                        </button>
                        <span>
                            Page {currentPage} of {totalPages}
                        </span>
                        <button
                            onClick={() => handlePageChange(currentPage + 1)}
                            disabled={currentPage === totalPages}
                        >
                            Next
                        </button>
                    </div>
                </>
            )}
        </div>
    );
}
