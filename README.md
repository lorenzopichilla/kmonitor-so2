#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/sched.h>
#include <linux/sched/signal.h>
#include <linux/mm.h>
#include <linux/sysinfo.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("René Ornelis");
MODULE_DESCRIPTION("K Monitor — Monitor de salud del sistema en tiempo real");
MODULE_VERSION("1.0");

#define PROC_FILENAME "kmonitor_grupo1"

static struct proc_dir_entry *proc_entry;

static const char *kmonitor_get_process_state(long state)
{
    if (state == TASK_RUNNING)
        return "Running";
    if (state == TASK_INTERRUPTIBLE || state == TASK_UNINTERRUPTIBLE)
        return "Sleeping";
    if (state == EXIT_ZOMBIE)
        return "Zombie";
    return "Other";
}

static int kmonitor_show(struct seq_file *m, void *v)
{
    struct sysinfo si;
    si_meminfo(&si);

    unsigned long total_kb  = (si.totalram  * si.mem_unit) / 1024;
    unsigned long free_kb   = (si.freeram   * si.mem_unit) / 1024;
    unsigned long used_kb   = total_kb - free_kb;
    unsigned long use_pct   = (total_kb > 0) ? (used_kb * 100 / total_kb) : 0;

    seq_printf(m, "============================================\n");
    seq_printf(m, "       K Monitor — Estado del Sistema       \n");
    seq_printf(m, "============================================\n\n");

    seq_printf(m, "[MEMORIA RAM]\n");
    seq_printf(m, "  Total   : %lu KB  (%lu MB)\n", total_kb, total_kb / 1024);
    seq_printf(m, "  Libre   : %lu KB  (%lu MB)\n", free_kb,  free_kb  / 1024);
    seq_printf(m, "  Usada   : %lu KB  (%lu MB)\n", used_kb,  used_kb  / 1024);
    seq_printf(m, "  Uso %%   : %lu%%\n\n",          use_pct);

    struct task_struct *p;

    seq_printf(m, "[PROCESOS ACTIVOS]\n");
    seq_printf(m, "  %-8s  %-20s  %-10s\n", "PID", "NOMBRE", "ESTADO");
    seq_printf(m, "  %-8s  %-20s  %-10s\n",
               "--------", "--------------------", "----------");

    rcu_read_lock();

    for_each_process(p) {
        seq_printf(m, "  %-8d  %-20s  %-10s\n",
                   p->pid,
                   p->comm,
                   kmonitor_get_process_state(p->__state));
    }

    rcu_read_unlock();

    seq_printf(m, "\n============================================\n");
    return 0;
}

static int kmonitor_open(struct inode *inode, struct file *file)
{
    return single_open(file, kmonitor_show, NULL);
}

static const struct proc_ops kmonitor_fops = {
    .proc_open    = kmonitor_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release
};

static int __init kmonitor_init(void)
{
    proc_entry = proc_create(PROC_FILENAME, 0444, NULL, &kmonitor_fops);

    if (!proc_entry) {
        pr_err("kmonitor: ERROR — no se pudo crear /proc/%s\n", PROC_FILENAME);
        return -ENOMEM;
    }

    pr_info("kmonitor: modulo cargado. Leer con: cat /proc/%s\n", PROC_FILENAME);
    return 0;
}

static void __exit kmonitor_exit(void)
{
    proc_remove(proc_entry);
    pr_info("kmonitor: modulo descargado. /proc/%s eliminado.\n", PROC_FILENAME);
}

module_init(kmonitor_init);
module_exit(kmonitor_exit);
