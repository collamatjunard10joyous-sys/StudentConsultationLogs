using System;
using System.IO;
using System.Collections.Generic;
using System.Linq;
using System.Text.RegularExpressions;
using System.Security.Cryptography;
using System.Text;

namespace StudentConsultationLogs
{
    class Program
    {
        static string folder        = "Data";
        static string recordsFile   = folder + "/records.txt";
        static string auditFile     = folder + "/audit.txt";
        static string reportsFolder = "Reports";

        static void Main(string[] args)
        {
            InitializeStorage();

            while (true)
            {
                Console.Clear();
                Console.WriteLine("===== STUDENT CONSULTATION LOG SYSTEM =====");
                Console.WriteLine("1. Add Record");
                Console.WriteLine("2. View Active Records");
                Console.WriteLine("3. Search Records");
                Console.WriteLine("4. Update Record");
                Console.WriteLine("5. Soft Delete Record (Deactivate)");
                Console.WriteLine("6. Hard Delete Record (Permanent)");
                Console.WriteLine("7. View Deleted Records");
                Console.WriteLine("8. Generate Report");
                Console.WriteLine("9. View Audit Log");
                Console.WriteLine("0. Exit");
                Console.Write("\nChoose: ");

                string choice = Console.ReadLine();

                try
                {
                    switch (choice)
                    {
                        case "1": AddRecord();          break;
                        case "2": ViewRecords();        break;
                        case "3": SearchRecords();      break;
                        case "4": UpdateRecord();       break;
                        case "5": SoftDeleteRecord();   break;
                        case "6": HardDeleteRecord();   break;
                        case "7": ViewDeletedRecords(); break;
                        case "8": GenerateReport();     break;
                        case "9": ViewAuditLog();       break;
                        case "0":
                            LogAudit("SYSTEM", "Application exited");
                            return;
                        default:
                            Console.WriteLine("Invalid choice.");
                            LogAudit("ERROR", "Invalid menu input: " + choice);
                            break;
                    }
                }
                catch (Exception ex)
                {
                    LogAudit("ERROR", ex.Message);
                    Console.WriteLine("Error: " + ex.Message);
                }

                Console.WriteLine("\nPress any key...");
                Console.ReadKey();
            }
        }

        static bool ValidateNotEmpty(string name, string studentId, string instructor,
                                     string course, string topic)
        {
            if (string.IsNullOrWhiteSpace(name)       ||
                string.IsNullOrWhiteSpace(studentId)  ||
                string.IsNullOrWhiteSpace(instructor) ||
                string.IsNullOrWhiteSpace(course)     ||
                string.IsNullOrWhiteSpace(topic))
            {
                Console.WriteLine("All fields are required.");
                LogAudit("ERROR", "Validation failed: empty field(s)");
                return false;
            }
            return true;
        }

        static bool ValidateStudentId(string studentId)
        {
            if (!Regex.IsMatch(studentId.Trim(), @"^\d{4}-\d{5}$"))
            {
                Console.WriteLine("Student ID must be in format YYYY-NNNNN (e.g. 2024-00123).");
                LogAudit("ERROR", "Validation failed: invalid student ID = " + studentId);
                return false;
            }
            return true;
        }

        static bool ValidateRecordId(string id)
        {
            if (string.IsNullOrWhiteSpace(id) || !id.StartsWith("SCL-"))
            {
                Console.WriteLine("Invalid Record ID format. Must start with SCL-");
                LogAudit("ERROR", "Validation failed: invalid record ID = " + id);
                return false;
            }
            return true;
        }

        static bool ValidateNotEmpty(string value, string fieldName)
        {
            if (string.IsNullOrWhiteSpace(value))
            {
                Console.WriteLine(fieldName + " cannot be empty.");
                LogAudit("ERROR", "Validation failed: " + fieldName + " is empty");
                return false;
            }
            return true;
        }

        static void InitializeStorage()
        {
            Directory.CreateDirectory(folder);
            Directory.CreateDirectory(reportsFolder);

            if (!File.Exists(recordsFile)) File.Create(recordsFile).Close();
            if (!File.Exists(auditFile))   File.Create(auditFile).Close();

            LogAudit("SYSTEM", "Storage initialized");
        }

        static List<string> LoadRecords()
        {
            return File.ReadAllLines(recordsFile).ToList();
        }

        static void SaveRecords(List<string> records)
        {
            File.WriteAllLines(recordsFile, records);
        }

        static void AppendRecord(string record)
        {
            File.AppendAllText(recordsFile, record + Environment.NewLine);
        }

        static string GenerateId()
        {
            var lines = LoadRecords();
            int max = 0;
            foreach (string line in lines)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var p = line.Split('|');
                if (p[0].StartsWith("SCL-"))
                {
                    int n;
                    if (int.TryParse(p[0].Substring(4), out n))
                        if (n > max) max = n;
                }
            }
            return "SCL-" + (max + 1).ToString("D6");
        }

        static string[] FindRecord(List<string> records, string id)
        {
            foreach (string line in records)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var p = line.Split('|');
                if (p[0] == id) return p;
            }
            return null;
        }

        static void LogAudit(string action, string details)
        {
            string log = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss") +
                         " | " + action + " | " + details;
            File.AppendAllText(auditFile, log + Environment.NewLine);
        }

        static void ViewAuditLog()
        {
            var lines = File.ReadAllLines(auditFile);
            Console.WriteLine("\n===== AUDIT LOG (Last 30 entries) =====");
            foreach (var line in lines.Skip(Math.Max(0, lines.Length - 30)))
                Console.WriteLine(line);
            Console.WriteLine("\nTotal entries: " + lines.Length);
            LogAudit("READ", "Viewed audit log");
        }

        static void GenerateReport()
        {
            var lines = LoadRecords();

            int total = 0, active = 0, inactive = 0;
            var byCourse     = new Dictionary<string, int>();
            var byInstructor = new Dictionary<string, int>();

            foreach (string line in lines)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var p = line.Split('|');
                if (p.Length < 10) continue;

                total++;
                if (p[8] == "True")
                {
                    active++;

                    if (byCourse.ContainsKey(p[4]))
                        byCourse[p[4]]++;
                    else
                        byCourse[p[4]] = 1;

                    if (byInstructor.ContainsKey(p[3]))
                        byInstructor[p[3]]++;
                    else
                        byInstructor[p[3]] = 1;
                }
                else
                {
                    inactive++;
                }
            }

            var sb = new StringBuilder();
            sb.AppendLine("===== STUDENT CONSULTATION REPORT =====");
            sb.AppendLine("Generated : " + DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"));
            sb.AppendLine();
            sb.AppendLine("Total Records    : " + total);
            sb.AppendLine("Active Records   : " + active);
            sb.AppendLine("Inactive Records : " + inactive);
            sb.AppendLine();
            sb.AppendLine("--- Consultations by Course ---");
            foreach (var kv in byCourse.OrderByDescending(x => x.Value))
                sb.AppendLine("  " + kv.Key + ": " + kv.Value);
            sb.AppendLine();
            sb.AppendLine("--- Consultations by Instructor ---");
            foreach (var kv in byInstructor.OrderByDescending(x => x.Value))
                sb.AppendLine("  " + kv.Key + ": " + kv.Value);

            string report = sb.ToString();
            Console.WriteLine("\n" + report);

            string reportPath = reportsFolder + "/report_" +
                DateTime.Now.ToString("yyyyMMdd_HHmmss") + ".txt";
            File.WriteAllText(reportPath, report);

            LogAudit("REPORT", "Generated report -> " + reportPath);
            Console.WriteLine("Report saved to: " + reportPath);
        }

        static void AddRecord()
        {
            Console.Write("Student Name      : "); string studentName = Console.ReadLine();
            Console.Write("Student ID        : "); string studentId   = Console.ReadLine();
            Console.Write("Instructor Name   : "); string instructor  = Console.ReadLine();
            Console.Write("Course            : "); string course      = Console.ReadLine();
            Console.Write("Consultation Topic: "); string topic       = Console.ReadLine();

            if (!ValidateNotEmpty(studentName, studentId, instructor, course, topic)) return;
            if (!ValidateStudentId(studentId)) return;

            string id       = GenerateId();
            string now      = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
            string checksum = ComputeChecksum(id, studentName.Trim(), studentId.Trim(),
                                              instructor.Trim(), course.Trim(), topic.Trim(), now);

            string record = string.Join("|",
                new string[] {
                    id, studentName.Trim(), studentId.Trim(),
                    instructor.Trim(), course.Trim(), topic.Trim(),
                    now, now, "True", checksum
                });

            AppendRecord(record);
            LogAudit("ADD", "RecordId=" + id + " Student=" + studentName.Trim());
            Console.WriteLine("Record added! ID: " + id);
        }

        static void ViewRecords()
        {
            var lines = LoadRecords();
            Console.WriteLine("\n===== ACTIVE RECORDS =====");
            bool any = false;

            foreach (string line in lines)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var p = line.Split('|');
                if (p.Length < 10 || p[8] == "False") continue;
                PrintRecord(p);
                any = true;
            }

            if (!any) Console.WriteLine("(No active records.)");
            LogAudit("READ", "Viewed all active records");
        }

        static void SearchRecords()
        {
            Console.WriteLine("Search by: 1) Student Name  2) Student ID  3) Instructor  4) Course");
            Console.Write("Choice: ");
            string by = Console.ReadLine();

            Console.Write("Keyword: ");
            string keyword = Console.ReadLine().ToLower().Trim();

            if (!ValidateNotEmpty(keyword, "Keyword")) return;

            var lines = LoadRecords();
            Console.WriteLine("\n===== SEARCH RESULTS =====");
            int count = 0;

            foreach (string line in lines)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var p = line.Split('|');
                if (p.Length < 10 || p[8] == "False") continue;

                bool match = false;
                switch (by)
                {
                    case "1": match = p[1].ToLower().Contains(keyword); break;
                    case "2": match = p[2].ToLower().Contains(keyword); break;
                    case "3": match = p[3].ToLower().Contains(keyword); break;
                    case "4": match = p[4].ToLower().Contains(keyword); break;
                    default:  match = false;                            break;
                }

                if (match) { PrintRecord(p); count++; }
            }

            if (count == 0) Console.WriteLine("No matching records found.");
            LogAudit("READ", "Search by=" + by + " keyword=" + keyword + " results=" + count);
        }

        static void UpdateRecord()
        {
            Console.Write("Enter Record ID to update: ");
            string id = Console.ReadLine().Trim();

            if (!ValidateRecordId(id)) return;

            var records = LoadRecords();
            bool found  = false;

            for (int i = 0; i < records.Count; i++)
            {
                if (string.IsNullOrWhiteSpace(records[i])) continue;
                var p = records[i].Split('|');
                if (p[0] != id) continue;

                found = true;
                Console.WriteLine("\nCurrent values:");
                PrintRecord(p);

                Console.Write("\nNew Topic (Enter to keep current)  : ");
                string newTopic = Console.ReadLine();
                Console.Write("New Course (Enter to keep current) : ");
                string newCourse = Console.ReadLine();

                if (!string.IsNullOrWhiteSpace(newTopic))  p[5] = newTopic.Trim();
                if (!string.IsNullOrWhiteSpace(newCourse)) p[4] = newCourse.Trim();

                p[7] = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
                p[9] = ComputeChecksum(p[0], p[1], p[2], p[3], p[4], p[5], p[6]);

                records[i] = string.Join("|", p);
                break;
            }

            if (found)
            {
                SaveRecords(records);
                LogAudit("UPDATE", "RecordId=" + id);
                Console.WriteLine("Record updated.");
            }
            else
            {
                Console.WriteLine("Record not found.");
                LogAudit("ERROR", "Update: Record not found ID=" + id);
            }
        }

        static void SoftDeleteRecord()
        {
            Console.Write("Enter Record ID to deactivate: ");
            string id = Console.ReadLine().Trim();

            if (!ValidateRecordId(id)) return;

            var records = LoadRecords();
            bool found  = false;

            for (int i = 0; i < records.Count; i++)
            {
                if (string.IsNullOrWhiteSpace(records[i])) continue;
                var p = records[i].Split('|');
                if (p[0] != id) continue;

                if (p[8] == "False") { Console.WriteLine("Record is already inactive."); return; }

                found  = true;
                p[8]   = "False";
                p[7]   = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
                p[9]   = ComputeChecksum(p[0], p[1], p[2], p[3], p[4], p[5], p[6]);
                records[i] = string.Join("|", p);
                break;
            }

            if (found)
            {
                SaveRecords(records);
                LogAudit("DELETE", "SOFT RecordId=" + id);
                Console.WriteLine("Record deactivated (soft deleted).");
            }
            else
            {
                Console.WriteLine("Record not found.");
                LogAudit("ERROR", "SoftDelete: Record not found ID=" + id);
            }
        }

        static void HardDeleteRecord()
        {
            Console.Write("Enter Record ID to permanently delete: ");
            string id = Console.ReadLine().Trim();

            if (!ValidateRecordId(id)) return;

            var records = LoadRecords();
            int removed = records.RemoveAll(delegate(string line)
            {
                if (string.IsNullOrWhiteSpace(line)) return false;
                return line.Split('|')[0] == id;
            });

            if (removed > 0)
            {
                SaveRecords(records);
                LogAudit("DELETE", "HARD RecordId=" + id);
                Console.WriteLine("Record permanently deleted.");
            }
            else
            {
                Console.WriteLine("Record not found.");
                LogAudit("ERROR", "HardDelete: Record not found ID=" + id);
            }
        }

        static void ViewDeletedRecords()
        {
            var lines = LoadRecords();
            Console.WriteLine("\n===== INACTIVE / DELETED RECORDS =====");
            bool any = false;

            foreach (string line in lines)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                var p = line.Split('|');
                if (p.Length < 10 || p[8] == "True") continue;
                PrintRecord(p);
                any = true;
            }

            if (!any) Console.WriteLine("(No inactive records.)");
            LogAudit("READ", "Viewed deleted/inactive records");
        }

        static string ComputeChecksum(params string[] fields)
        {
            string raw = string.Join("|", fields);
            byte[] inputBytes = Encoding.UTF8.GetBytes(raw);
            byte[] hash;
            using (MD5 md5 = MD5.Create())
            {
                hash = md5.ComputeHash(inputBytes);
            }
            return BitConverter.ToString(hash).Replace("-", "").ToLower();
        }

        static void PrintRecord(string[] p)
        {
            Console.WriteLine("--------------------------------");
            Console.WriteLine("Record ID   : " + p[0]);
            Console.WriteLine("Student     : " + p[1] + " (" + p[2] + ")");
            Console.WriteLine("Instructor  : " + p[3]);
            Console.WriteLine("Course      : " + p[4]);
            Console.WriteLine("Topic       : " + p[5]);
            Console.WriteLine("Created At  : " + p[6]);
            Console.WriteLine("Updated At  : " + p[7]);
            Console.WriteLine("Active      : " + p[8]);
            Console.WriteLine("Checksum    : " + p[9]);
        }
    }
}
