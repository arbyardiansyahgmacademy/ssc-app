# KODE.GS — BACKEND GOOGLE APPS SCRIPT (SSC v2.2)

File ini berisi kode lengkap backend Google Apps Script untuk sistem **Student Service Center (SSC)**.
Kode ini mengelola sinkronisasi Google Sheets, penyimpanan arsip dokumen PDF di Google Drive, serta endpoint API (REST JSON) untuk frontend Blogger.

---

```javascript
/**
 * ==========================================================================
 * STUDENT SERVICE CENTER (SSC) — GOOGLE APPS SCRIPT BACKEND
 * Version: 2.2 (Full Sync Support)
 * ==========================================================================
 */

const SHEET_NAMES = {
  USERS: "Users",
  STUDENTS: "Students",
  SERVICES: "Services",
  SUBMISSIONS: "Submissions",
  DOCUMENTS: "Documents",
  NOTIFICATIONS: "Notifications",
  ACTIVITY_LOGS: "Activity_Logs"
};

const FOLDER_ATTACHMENTS = "SSC_Attachments";
const FOLDER_GENERATED_PDF = "SSC_Generated_PDF";
const FALLBACK_PDF_URL = "https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf";
const TIMEZONE = "GMT+7";

function doGet(e) {
  return respond({ status: "online", platform: "SSC Backend", version: "2.2", timestamp: new Date().toISOString() });
}

function doPost(e) {
  const lock = LockService.getScriptLock();
  try {
    lock.waitLock(20000);
    let payload = {};
    if (e && e.postData && e.postData.contents) {
      try { payload = JSON.parse(e.postData.contents); } catch (err) {
        return respond({ status: "error", message: "Format JSON tidak valid." });
      }
    }
    const action = payload.action;
    let result;
    switch (action) {
      case "login": result = handleUserLogin(payload.username, payload.password); break;
      case "getPublicServices":
      case "getServicesList": result = getServicesList(); break;
      case "createSubmission": result = createNewSubmission(payload.data); break;
      case "getStudentSubmissions": result = getStudentSubmissions(payload.student_id); break;
      case "getAllSubmissions": result = getAllSubmissions(); break;
      case "getSubmissionDetail": result = getSubmissionDetail(payload.submission_id); break;
      case "trackSubmission": result = trackSubmissionByTicket(payload.ticket || payload.submission_id); break;
      case "updateSubmissionStatus":
        result = updateSubmissionStatus(payload.submission_id, payload.status, payload.note, payload.admin_user);
        break;
      case "getStudentsList": result = getStudentsList(); break;
      case "getStudentProfile": result = getStudentProfile(payload.student_id); break;
      case "getStudentDocuments": result = getStudentDocuments(payload.student_id); break;
      case "getAllDocuments": result = getAllDocuments(); break;
      case "getNotifications": result = getNotifications(payload.recipient_id); break;
      case "getAllNotifications": result = getAllNotifications(); break;
      case "markNotificationRead": result = markNotificationRead(payload.notification_id); break;
      case "getActivityLogs": result = getActivityLogs(payload.limit); break;
      case "uploadAttachment":
        result = uploadAttachmentToDrive(payload.filename, payload.mimeType, payload.base64Data, payload.submission_id);
        break;
      case "syncAll": result = syncAllData(); break;
      case "ping": result = { status: "success", message: "pong" }; break;
      default: result = { status: "error", message: `Action '${action}' tidak didukung.` };
    }
    return respond(result);
  } catch (error) {
    return respond({ status: "error", message: "Kesalahan server: " + error.toString() });
  } finally {
    try { lock.releaseLock(); } catch (e) {}
  }
}

function respond(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON);
}

/* ==========================================================================
   SYNC ALL — endpoint utama untuk frontend
   ========================================================================== */
function syncAllData() {
  try {
    return {
      status: "success",
      data: {
        services: (getServicesList().data) || [],
        students: (getStudentsList().data) || [],
        submissions: (getAllSubmissions().data) || [],
        documents: (getAllDocuments().data) || [],
        notifications: (getAllNotifications().data) || [],
        activity_logs: (getActivityLogs(200).data) || []
      }
    };
  } catch (err) {
    return { status: "error", message: "Sync gagal: " + err.toString() };
  }
}

/* ==========================================================================
   SETUP DATABASE
   ========================================================================== */
function setupDatabase() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();

  let shUsers = ss.getSheetByName(SHEET_NAMES.USERS) || ss.insertSheet(SHEET_NAMES.USERS);
  if (shUsers.getLastRow() === 0) {
    shUsers.appendRow(["user_id","username","password","role","name","student_id","status","created_at"]);
    shUsers.getRange(1,1,1,8).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
    shUsers.appendRow(["USR-001","admin","admin2026","admin","Drs. H. Mulyadi, M.Pd","","active", fmt(new Date())]);
    shUsers.appendRow(["USR-002","peserta","edudigital","student","Arbi Pratama","STD-2026-001","active", fmt(new Date())]);
  }

  let shStudents = ss.getSheetByName(SHEET_NAMES.STUDENTS) || ss.insertSheet(SHEET_NAMES.STUDENTS);
  if (shStudents.getLastRow() === 0) {
    shStudents.appendRow(["student_id","nis","nisn","name","class","major","email","phone","status","created_at"]);
    shStudents.getRange(1,1,1,10).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
    shStudents.appendRow(["STD-2026-001","2122101","0051289110","Arbi Pratama","XII Rekayasa Perangkat Lunak 1","RPL","arbi.pratama@student.sch.id","081234567890","Aktif", fmt(new Date())]);
    shStudents.appendRow(["STD-2026-002","2122102","0051289111","Siti Rahmawati","XII Rekayasa Perangkat Lunak 1","RPL","siti.rahma@student.sch.id","081298765432","Aktif", fmt(new Date())]);
    shStudents.appendRow(["STD-2026-003","2122201","0051289112","Budi Santoso","XI Teknik Komputer & Jaringan 2","TKJ","budi.santoso@student.sch.id","085678123456","Aktif", fmt(new Date())]);
  }

  let shServices = ss.getSheetByName(SHEET_NAMES.SERVICES) || ss.insertSheet(SHEET_NAMES.SERVICES);
  if (shServices.getLastRow() === 0) {
    shServices.appendRow(["service_id","service_name","category","description","requirements","template_id","estimated_days","status"]);
    shServices.getRange(1,1,1,8).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
    [
      ["SRV-AKTIF","Surat Keterangan Siswa Aktif","Administrasi","Menerangkan bahwa siswa bersangkutan aktif menempuh pendidikan pada tahun ajaran berjalan untuk keperluan tunjangan orang tua / perbankan.","Scan Kartu Pelajar & Kartu Keluarga","DOC_TEMPLATE_AKTIF_01",1,"active"],
      ["SRV-MAGANG","Surat Pengantar PKL / Magang","Magang","Surat permohonan resmi penempatan magang ke instansi, industri, atau Dunia Usaha/Dunia Industri (DU/DI).","Nama Perusahaan, Alamat, PIC & Periode Magang","DOC_TEMPLATE_MAGANG_02",2,"active"],
      ["SRV-REKOM","Surat Rekomendasi Beasiswa","Rekomendasi","Rekomendasi resmi pihak sekolah untuk pengajuan beasiswa prestasi akademik maupun non-akademik.","Bukti Prestasi / Formulir Beasiswa Terkait","DOC_TEMPLATE_REKOM_03",2,"active"],
      ["SRV-LEGALISIR","Permohonan Legalisir Dokumen","Akademik","Pengesahan salinan rapor, ijazah, atau sertifikat kompetensi siswa.","Scan Dokumen Asli yang akan dilegalisir","DOC_TEMPLATE_LEGALISIR_04",2,"active"],
      ["SRV-IZIN","Surat Dispensasi / Izin Khusus","Administrasi","Dispensasi meninggalkan KBM untuk mengikuti perlombaan resmi, kejuaraan, atau tugas sekolah.","Surat Undangan Lomba / Penugasan Resmi","DOC_TEMPLATE_IZIN_05",1,"active"]
    ].forEach(r => shServices.appendRow(r));
  }

  let shSubmissions = ss.getSheetByName(SHEET_NAMES.SUBMISSIONS) || ss.insertSheet(SHEET_NAMES.SUBMISSIONS);
  if (shSubmissions.getLastRow() === 0) {
    shSubmissions.appendRow(["submission_id","student_id","student_name","student_class","service_id","service_name","submission_date","purpose","attachment_url","status","admin_note","document_url","updated_at"]);
    shSubmissions.getRange(1,1,1,13).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
    shSubmissions.getRange("A:A").setNumberFormat("@");
  }

  let shDocs = ss.getSheetByName(SHEET_NAMES.DOCUMENTS) || ss.insertSheet(SHEET_NAMES.DOCUMENTS);
  if (shDocs.getLastRow() === 0) {
    shDocs.appendRow(["document_id","submission_id","document_name","service_name","drive_file_id","pdf_url","created_at"]);
    shDocs.getRange(1,1,1,7).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
  }

  let shNotifs = ss.getSheetByName(SHEET_NAMES.NOTIFICATIONS) || ss.insertSheet(SHEET_NAMES.NOTIFICATIONS);
  if (shNotifs.getLastRow() === 0) {
    shNotifs.appendRow(["notification_id","recipient_id","title","message","type","is_read","created_at"]);
    shNotifs.getRange(1,1,1,7).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
  }

  let shLogs = ss.getSheetByName(SHEET_NAMES.ACTIVITY_LOGS) || ss.insertSheet(SHEET_NAMES.ACTIVITY_LOGS);
  if (shLogs.getLastRow() === 0) {
    shLogs.appendRow(["log_id","user","role","action","description","timestamp"]);
    shLogs.getRange(1,1,1,6).setFontWeight("bold").setBackground("#0f2744").setFontColor("#ffffff");
    shLogs.appendRow(["LOG-0000","SYSTEM","System","Setup Database","Inisialisasi skema basis data SSC selesai.", fmtFull(new Date())]);
  }

  return "✅ Setup Database Selesai!";
}

function fmt(d) { return Utilities.formatDate(d, TIMEZONE, "yyyy-MM-dd HH:mm"); }
function fmtFull(d) { return Utilities.formatDate(d, TIMEZONE, "yyyy-MM-dd HH:mm:ss"); }

/* ==========================================================================
   AUTHENTICATION
   ========================================================================== */
function handleUserLogin(username, password) {
  if (!username || !password) return { status: "error", message: "Username dan password wajib diisi." };
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const shUsers = ss.getSheetByName(SHEET_NAMES.USERS);
  if (!shUsers) return { status: "error", message: "Sheet Users belum dibuat." };
  const data = shUsers.getDataRange().getValues();
  const inputU = String(username).trim().toLowerCase();
  const inputP = String(password).trim();

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const u = String(row[1]).trim();
    const p = String(row[2]).trim();
    const role = String(row[3]).trim();
    const name = String(row[4]).trim();
    const studentId = String(row[5] || "").trim();
    const status = String(row[6] || "").trim().toLowerCase();

    if (u.toLowerCase() === inputU && p === inputP) {
      if (status && status !== "active") return { status: "error", message: "Akun Anda nonaktif." };
      let resolvedStudentId = studentId;
      if (role === "student" && !resolvedStudentId) {
        resolvedStudentId = findStudentIdByUsername(u) || "STD-2026-001";
      }
      logActivity(u, role, "Login", "Pengguna berhasil masuk.");
      return {
        status: "success",
        data: { user_id: row[0], username: u, role: role, name: name, student_id: resolvedStudentId }
      };
    }
  }
  return { status: "error", message: "Username atau Password salah." };
}

function findStudentIdByUsername(username) {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sh = ss.getSheetByName(SHEET_NAMES.STUDENTS);
    if (!sh) return null;
    const data = sh.getDataRange().getValues();
    const target = String(username).trim().toLowerCase();
    for (let i = 1; i < data.length; i++) {
      if (String(data[i][1]).trim().toLowerCase() === target || String(data[i][6]).trim().toLowerCase() === target) {
        return String(data[i][0]).trim();
      }
    }
  } catch (e) {}
  return null;
}

/* ==========================================================================
   SERVICES
   ========================================================================== */
function getServicesList() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.SERVICES);
  if (!sh) return { status: "error", message: "Sheet Services tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const services = [];
  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    if (!row[0]) continue;
    if (String(row[7] || "").toLowerCase() !== "active") continue;
    services.push({
      service_id: String(row[0]),
      service_name: String(row[1]),
      category: String(row[2]),
      description: String(row[3]),
      requirements: String(row[4]),
      template_id: String(row[5] || ""),
      estimated_days: Number(row[6]) || 1,
      status: "active"
    });
  }
  return { status: "success", data: services };
}

/* ==========================================================================
   SUBMISSIONS
   ========================================================================== */
function createNewSubmission(subData) {
  if (!subData || !subData.student_id || !subData.service_id) {
    return { status: "error", message: "Data pengajuan tidak lengkap." };
  }
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.SUBMISSIONS);
  if (!sh) return { status: "error", message: "Sheet Submissions tidak ditemukan." };

  let submissionId = String(subData.submission_id || "").trim();
  if (!submissionId) submissionId = generateNextTicketNumber();

  const nowStr = fmt(new Date());
  const record = {
    submission_id: submissionId,
    student_id: subData.student_id || "",
    student_name: subData.student_name || "",
    student_class: subData.student_class || "",
    service_id: subData.service_id || "",
    service_name: subData.service_name || "",
    submission_date: subData.submission_date || nowStr,
    purpose: subData.purpose || "",
    attachment_url: subData.attachment_url || "",
    status: subData.status || "MENUNGGU",
    admin_note: subData.admin_note || "",
    document_url: subData.document_url || "",
    updated_at: subData.updated_at || nowStr
  };

  sh.appendRow([
    record.submission_id, record.student_id, record.student_name, record.student_class,
    record.service_id, record.service_name, record.submission_date, record.purpose,
    record.attachment_url, record.status, record.admin_note, record.document_url, record.updated_at
  ]);

  logActivity(record.student_name, "Student", "Create Submission", `Pengajuan ${submissionId} (${record.service_name}) dibuat.`);
  createNotification(record.student_id, "Pengajuan Diterima",
    `Pengajuan ${submissionId} (${record.service_name}) telah kami terima.`, "info");

  return { status: "success", message: "Pengajuan berhasil dikirimkan!", submission_id: submissionId, data: record };
}

function generateNextTicketNumber() {
  const props = PropertiesService.getScriptProperties();
  const year = new Date().getFullYear();
  const key = `SSC_COUNTER_${year}`;
  let counter = Number(props.getProperty(key) || 0) + 1;
  props.setProperty(key, String(counter));
  return `SSC-${year}-${("00000" + counter).slice(-5)}`;
}

function getStudentSubmissions(studentId) {
  if (!studentId) return { status: "error", message: "student_id wajib diisi." };
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.SUBMISSIONS);
  if (!sh) return { status: "error", message: "Sheet Submissions tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    if (String(data[i][1]).trim() !== String(studentId).trim()) continue;
    list.push(rowToSubmissionObject(data[i]));
  }
  list.sort((a, b) => String(b.submission_date).localeCompare(String(a.submission_date)));
  return { status: "success", data: list };
}

function getAllSubmissions() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.SUBMISSIONS);
  if (!sh) return { status: "error", message: "Sheet Submissions tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    list.push(rowToSubmissionObject(data[i]));
  }
  list.sort((a, b) => String(b.submission_date).localeCompare(String(a.submission_date)));
  return { status: "success", data: list };
}

function getSubmissionDetail(submissionId) {
  const found = findSubmissionRow(submissionId);
  if (!found) return { status: "error", message: "Pengajuan tidak ditemukan." };
  return { status: "success", data: rowToSubmissionObject(found.row) };
}

function trackSubmissionByTicket(ticket) {
  if (!ticket) return { status: "error", message: "Nomor tiket wajib diisi." };
  const found = findSubmissionRow(String(ticket).trim().toUpperCase());
  if (!found) return { status: "error", message: "Nomor tiket tidak ditemukan." };
  return { status: "success", data: rowToSubmissionObject(found.row) };
}

function findSubmissionRow(submissionId) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.SUBMISSIONS);
  if (!sh) return null;
  const data = sh.getDataRange().getValues();
  const target = String(submissionId).trim().toUpperCase();
  for (let i = 1; i < data.length; i++) {
    if (String(data[i][0]).trim().toUpperCase() === target) {
      return { row: data[i], rowIndex: i + 1, sheet: sh };
    }
  }
  return null;
}

function rowToSubmissionObject(row) {
  return {
    submission_id: String(row[0]),
    student_id: String(row[1]),
    student_name: String(row[2]),
    student_class: String(row[3]),
    service_id: String(row[4]),
    service_name: String(row[5]),
    submission_date: String(row[6]),
    purpose: String(row[7]),
    attachment_url: String(row[8] || ""),
    status: String(row[9] || "MENUNGGU"),
    admin_note: String(row[10] || ""),
    document_url: String(row[11] || ""),
    updated_at: String(row[12] || "")
  };
}

function updateSubmissionStatus(submissionId, newStatus, adminNote, adminUser) {
  if (!submissionId || !newStatus) return { status: "error", message: "Parameter tidak lengkap." };
  const found = findSubmissionRow(submissionId);
  if (!found) return { status: "error", message: "Pengajuan tidak ditemukan." };

  const sh = found.sheet;
  const rowIndex = found.rowIndex;
  const rowData = found.row;
  const nowStr = fmt(new Date());

  let documentUrl = rowData[11] || "";
  let driveFileId = "";

  if (newStatus === "SELESAI" && !documentUrl) {
    const pdfResult = generateOfficialDocumentPdf(submissionId, rowData);
    documentUrl = pdfResult.url;
    driveFileId = pdfResult.fileId;
    saveDocumentRecord(submissionId, `${rowData[5]} - ${rowData[2]}`, String(rowData[5]), driveFileId, documentUrl);
    createNotification(String(rowData[1]), "Surat Selesai Diterbitkan",
      `Pengajuan ${submissionId} (${rowData[5]}) telah disetujui & PDF siap diunduh.`, "success");
  } else {
    const notifMap = {
      "DIPROSES": ["Pengajuan Sedang Diproses", "info"],
      "PERLU REVISI": ["Pengajuan Perlu Revisi", "warning"],
      "DITOLAK": ["Pengajuan Ditolak", "error"]
    };
    if (notifMap[newStatus]) {
      createNotification(String(rowData[1]), notifMap[newStatus][0],
        `Pengajuan ${submissionId}: ${adminNote || newStatus}`, notifMap[newStatus][1]);
    }
  }

  sh.getRange(rowIndex, 10).setValue(newStatus);
  sh.getRange(rowIndex, 11).setValue(adminNote || "");
  sh.getRange(rowIndex, 12).setValue(documentUrl);
  sh.getRange(rowIndex, 13).setValue(nowStr);

  logActivity(adminUser || "Admin TU", "Admin", `Status → ${newStatus}`, `Tiket ${submissionId} diubah menjadi ${newStatus}.`);

  return { status: "success", message: `Pengajuan ${submissionId} diperbarui ke ${newStatus}.`, document_url: documentUrl };
}

/* ==========================================================================
   PDF GENERATOR
   ========================================================================== */
function generateOfficialDocumentPdf(submissionId, submissionRow) {
  try {
    const studentName = String(submissionRow[2]);
    const studentClass = String(submissionRow[3]);
    const serviceName = String(submissionRow[5]);
    const purpose = String(submissionRow[7]);
    const dateNow = Utilities.formatDate(new Date(), TIMEZONE, "dd MMMM yyyy");
    const targetFolder = getOrCreateFolder(FOLDER_GENERATED_PDF);

    const doc = DocumentApp.create(`SSC_Doc_${submissionId}`);
    const body = doc.getBody();
    body.setMarginTop(60).setMarginBottom(60).setMarginLeft(72).setMarginRight(72);

    body.appendParagraph("PEMERINTAH DAERAH PROVINSI JAWA TIMUR").setHeading(DocumentApp.ParagraphHeading.HEADING3).setAlignment(DocumentApp.HorizontalAlignment.CENTER);
    body.appendParagraph("DINAS PENDIDIKAN DAN KEBUDAYAAN").setHeading(DocumentApp.ParagraphHeading.HEADING2).setAlignment(DocumentApp.HorizontalAlignment.CENTER);
    body.appendParagraph("SEKOLAH MENENGAH KEJURUAN NEGERI 1").setHeading(DocumentApp.ParagraphHeading.HEADING2).setAlignment(DocumentApp.HorizontalAlignment.CENTER);
    body.appendParagraph("STUDENT SERVICE CENTER — DIGITAL ADMINISTRATION PLATFORM").setAlignment(DocumentApp.HorizontalAlignment.CENTER);
    body.appendHorizontalRule();

    body.appendParagraph("\n" + serviceName.toUpperCase()).setHeading(DocumentApp.ParagraphHeading.HEADING2).setAlignment(DocumentApp.HorizontalAlignment.CENTER);
    body.appendParagraph(`Nomor Registrasi: ${submissionId}\n`).setAlignment(DocumentApp.HorizontalAlignment.CENTER);

    body.appendParagraph("Yang bertanda tangan di bawah ini, Kepala Bagian Pelayanan Administrasi Kesiswaan, menerangkan dengan sesungguhnya bahwa:");
    body.appendParagraph(`Nama Lengkap      : ${studentName}`);
    body.appendParagraph(`Tingkat / Kelas   : ${studentClass}`);
    body.appendParagraph(`Keperluan         : ${purpose}`);
    body.appendParagraph(`Tanggal Terbit    : ${dateNow}\n`);
    body.appendParagraph("Demikian surat keterangan ini diterbitkan secara sah dan digital melalui platform Student Service Center (SSC).\n\n");
    body.appendParagraph(`Diterbitkan secara digital pada: ${dateNow}`).setAlignment(DocumentApp.HorizontalAlignment.RIGHT);
    body.appendParagraph("Kepala Unit Administrasi Kesiswaan").setAlignment(DocumentApp.HorizontalAlignment.RIGHT);
    body.appendParagraph("\n\n").setAlignment(DocumentApp.HorizontalAlignment.RIGHT);
    body.appendParagraph("( Ditetapkan Secara Elektronik )").setAlignment(DocumentApp.HorizontalAlignment.RIGHT);

    doc.saveAndClose();

    const pdfBlob = doc.getAs("application/pdf");
    const fileName = `${submissionId}_${serviceName.replace(/[^a-zA-Z0-9]/g, "_")}.pdf`;
    const pdfFile = targetFolder.createFile(pdfBlob).setName(fileName);
    pdfFile.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);

    try { DriveApp.getFileById(doc.getId()).setTrashed(true); } catch (e) {}

    return { url: pdfFile.getUrl(), fileId: pdfFile.getId() };
  } catch (err) {
    return { url: FALLBACK_PDF_URL, fileId: "", error: err.toString() };
  }
}

function getOrCreateFolder(name) {
  const folders = DriveApp.getFoldersByName(name);
  return folders.hasNext() ? folders.next() : DriveApp.createFolder(name);
}

/* ==========================================================================
   DOCUMENTS
   ========================================================================== */
function saveDocumentRecord(submissionId, documentName, serviceName, driveFileId, pdfUrl) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.DOCUMENTS);
  if (!sh) return;
  const docId = "DOC-" + ("0000" + sh.getLastRow()).slice(-4);
  sh.appendRow([docId, submissionId, documentName, serviceName, driveFileId || "", pdfUrl, fmt(new Date())]);
}

function getStudentDocuments(studentId) {
  if (!studentId) return { status: "error", message: "student_id wajib diisi." };
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const shSub = ss.getSheetByName(SHEET_NAMES.SUBMISSIONS);
  const shDoc = ss.getSheetByName(SHEET_NAMES.DOCUMENTS);
  if (!shSub || !shDoc) return { status: "error", message: "Sheet tidak ditemukan." };

  const subData = shSub.getDataRange().getValues();
  const myIds = new Set();
  for (let i = 1; i < subData.length; i++) {
    if (String(subData[i][1]).trim() === String(studentId).trim()) myIds.add(String(subData[i][0]).trim());
  }

  const docData = shDoc.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < docData.length; i++) {
    if (!docData[i][0]) continue;
    if (!myIds.has(String(docData[i][1]).trim())) continue;
    list.push({
      document_id: String(docData[i][0]),
      submission_id: String(docData[i][1]),
      document_name: String(docData[i][2]),
      service_name: String(docData[i][3]),
      drive_file_id: String(docData[i][4]),
      pdf_url: String(docData[i][5]),
      created_at: String(docData[i][6])
    });
  }
  list.sort((a, b) => String(b.created_at).localeCompare(String(a.created_at)));
  return { status: "success", data: list };
}

function getAllDocuments() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.DOCUMENTS);
  if (!sh) return { status: "error", message: "Sheet Documents tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    list.push({
      document_id: String(data[i][0]),
      submission_id: String(data[i][1]),
      document_name: String(data[i][2]),
      service_name: String(data[i][3]),
      drive_file_id: String(data[i][4]),
      pdf_url: String(data[i][5]),
      created_at: String(data[i][6])
    });
  }
  return { status: "success", data: list };
}

/* ==========================================================================
   NOTIFICATIONS
   ========================================================================== */
function createNotification(recipientId, title, message, type) {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sh = ss.getSheetByName(SHEET_NAMES.NOTIFICATIONS);
    if (!sh) return;
    const notifId = "NOTIF-" + ("0000" + sh.getLastRow()).slice(-4);
    sh.appendRow([notifId, recipientId, title, message, type || "info", "false", fmt(new Date())]);
  } catch (e) {}
}

function getNotifications(recipientId) {
  if (!recipientId) return { status: "error", message: "recipient_id wajib diisi." };
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.NOTIFICATIONS);
  if (!sh) return { status: "error", message: "Sheet Notifications tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    if (String(data[i][1]).trim() !== String(recipientId).trim()) continue;
    list.push({
      notification_id: String(data[i][0]),
      recipient_id: String(data[i][1]),
      title: String(data[i][2]),
      message: String(data[i][3]),
      type: String(data[i][4] || "info"),
      is_read: String(data[i][5]).toLowerCase() === "true",
      created_at: String(data[i][6])
    });
  }
  list.sort((a, b) => String(b.created_at).localeCompare(String(a.created_at)));
  return { status: "success", data: list };
}

function getAllNotifications() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.NOTIFICATIONS);
  if (!sh) return { status: "error", message: "Sheet Notifications tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    list.push({
      notification_id: String(data[i][0]),
      recipient_id: String(data[i][1]),
      title: String(data[i][2]),
      message: String(data[i][3]),
      type: String(data[i][4] || "info"),
      is_read: String(data[i][5]).toLowerCase() === "true",
      created_at: String(data[i][6])
    });
  }
  list.sort((a, b) => String(b.created_at).localeCompare(String(a.created_at)));
  return { status: "success", data: list };
}

function markNotificationRead(notificationId) {
  if (!notificationId) return { status: "error", message: "notification_id wajib diisi." };
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.NOTIFICATIONS);
  if (!sh) return { status: "error", message: "Sheet Notifications tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (String(data[i][0]) === String(notificationId)) {
      sh.getRange(i + 1, 6).setValue("true");
      return { status: "success", message: "Notifikasi ditandai dibaca." };
    }
  }
  return { status: "error", message: "Notifikasi tidak ditemukan." };
}

/* ==========================================================================
   STUDENTS
   ========================================================================== */
function getStudentsList() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.STUDENTS);
  if (!sh) return { status: "error", message: "Sheet Students tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    list.push({
      student_id: String(data[i][0]),
      nis: String(data[i][1]),
      nisn: String(data[i][2]),
      name: String(data[i][3]),
      class: String(data[i][4]),
      major: String(data[i][5]),
      email: String(data[i][6]),
      phone: String(data[i][7]),
      status: String(data[i][8] || "Aktif")
    });
  }
  return { status: "success", data: list };
}

function getStudentProfile(studentId) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.STUDENTS);
  if (!sh) return { status: "error", message: "Sheet Students tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  for (let i = 1; i < data.length; i++) {
    if (String(data[i][0]).trim() === String(studentId).trim()) {
      return {
        status: "success",
        data: {
          student_id: String(data[i][0]),
          nis: String(data[i][1]),
          nisn: String(data[i][2]),
          name: String(data[i][3]),
          class: String(data[i][4]),
          major: String(data[i][5]),
          email: String(data[i][6]),
          phone: String(data[i][7]),
          status: String(data[i][8] || "Aktif")
        }
      };
    }
  }
  return { status: "error", message: "Siswa tidak ditemukan." };
}

/* ==========================================================================
   ACTIVITY LOGS
   ========================================================================== */
function logActivity(user, role, action, description) {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sh = ss.getSheetByName(SHEET_NAMES.ACTIVITY_LOGS);
    if (!sh) return;
    const logId = "LOG-" + ("0000" + sh.getLastRow()).slice(-4);
    sh.appendRow([logId, user, role, action, description, fmtFull(new Date())]);
  } catch (e) {}
}

function getActivityLogs(limit) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sh = ss.getSheetByName(SHEET_NAMES.ACTIVITY_LOGS);
  if (!sh) return { status: "error", message: "Sheet Activity_Logs tidak ditemukan." };
  const data = sh.getDataRange().getValues();
  const list = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;
    list.push({
      log_id: String(data[i][0]),
      user: String(data[i][1]),
      role: String(data[i][2]),
      action: String(data[i][3]),
      description: String(data[i][4]),
      timestamp: String(data[i][5])
    });
  }
  list.sort((a, b) => String(b.timestamp).localeCompare(String(a.timestamp)));
  return { status: "success", data: list.slice(0, Number(limit) || 100) };
}

/* ==========================================================================
   UPLOAD
   ========================================================================== */
function uploadAttachmentToDrive(filename, mimeType, base64Data, submissionId) {
  try {
    if (!base64Data) return { status: "error", message: "Data file kosong." };
    const folder = getOrCreateFolder(FOLDER_ATTACHMENTS);
    const decoded = Utilities.base64Decode(base64Data);
    const blob = Utilities.newBlob(decoded, mimeType || "application/octet-stream", filename || "attachment");
    const file = folder.createFile(blob);
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
    return { status: "success", file_id: file.getId(), file_url: file.getUrl(), download_url: "https://drive.google.com/uc?export=download&id=" + file.getId() };
  } catch (err) {
    return { status: "error", message: "Upload gagal: " + err.toString() };
  }
}

/* ==========================================================================
   TEST FUNCTIONS
   ========================================================================== */
function TEST_setup() { Logger.log(setupDatabase()); }
function TEST_sync() { Logger.log(JSON.stringify(syncAllData()).substring(0, 500)); }
function TEST_login() { Logger.log(handleUserLogin("admin", "admin2026")); }
```
