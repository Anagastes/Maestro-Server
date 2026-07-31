# Log Rotation Setup

## Problem Solved
- **Issue**: MPD logs grew to 126GB (`/var/log/mpd/*.log`), consuming entire disk
- **Root Cause**: No log rotation configured; logs accumulated indefinitely
- **Solution**: Implemented logrotate configuration with automatic daily rotation

## Configuration

### MPD Logrotate Config
File: `/etc/logrotate.d/mpd`

```
/var/log/mpd/*.log {
    daily              # Rotate daily
    rotate 7           # Keep 7 days of logs
    compress           # Compress rotated logs (gzip)
    delaycompress      # Compress the previous rotation, not the current
    missingok          # Don't error if log file is missing
    notifempty         # Don't rotate empty files
    create 0640 mpd mpd  # Create new log file with MPD permissions
    sharedscripts      # Run postrotate script once for all files
    postrotate
        systemctl reload mpd > /dev/null 2>&1 || true
    endscript
}
```

### How It Works
1. **Daily**: Every day at 3:00 AM (cron), logrotate checks `/etc/logrotate.d/`
2. **Rotation**: Current log is renamed with date stamp, compressed
3. **Retention**: Keeps 7 compressed days (saves ~98% space: 18 MB compressed vs 18 GB uncompressed)
4. **Cleanup**: Files older than 7 days are automatically deleted
5. **Reload**: MPD is signaled to use new log file

### Expected Disk Savings
- **Before**: 126GB for single MPD log file
- **After**: ~126MB for 7 days of compressed logs
- **Savings**: ~125.9GB (99.9% reduction!)

## Deployment

Config is installed at: `/etc/logrotate.d/mpd`

Test with:
```bash
sudo logrotate -d /etc/logrotate.d/mpd  # Dry run
sudo logrotate -f /etc/logrotate.d/mpd  # Force rotation
```

## Export Temp Directory Fix

Additionally, fixed export temp directory priority to avoid tmpfs quota limits:

### Previous Order (Problematic)
1. `/tmp` (tmpfs, limited to ~4GB with user quota)
2. `/var/tmp`
3. `~/.cache/maestro/temp`
4. `~/maestro_temp`

### New Order (Fixed)
1. `~/.cache/maestro/temp` (main filesystem)
2. `~/maestro_temp` (main filesystem)
3. `/var/tmp` (main filesystem)
4. `/tmp` (tmpfs - last resort)

This prevents "Disk quota exceeded" errors when creating large ZIP exports (2.7GB+).

## Monitoring

Check current log usage:
```bash
du -sh /var/log/mpd/
ls -lh /var/log/mpd/ | tail
```

View compression ratio:
```bash
du -sh /var/log/mpd/*.log*
```

## Related Files
- [playlist_export.py](services/playlist_export.py) - Export temp directory selection logic
- Git commit: d312986 - "Fix: Prioritize main filesystem temp directories over tmpfs for exports"
