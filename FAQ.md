1. **Q: How is SLA calculated?**
   A: Based on priority. P1=4hrs, P2=8hrs, P3=24hrs, P4=72hrs from created_at.

2. **Q: What happens on SLA breach?**
   A: `sla_breach` flag = true, email sent to assignee + manager.

3. **Q: How does approval work?**
   A: All "Normal" changes need manager approval. Status stays "Authorize" until approved.

4. **Q: Can I change status workflow?**
   A: Yes, edit `Incident.java` enum. ServiceNow uses: New, In Progress, Resolved, Closed.

5. **Q: How to add new priority?**
   A: Add row in `sla_configs` table and update frontend dropdown.

... 15 more questions covering JWT expiry, email setup, adding custom fields, etc
