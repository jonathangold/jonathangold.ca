---
title: "Waiting for a Systemd Job to Complete"
date: 2018-05-15T00:56:21-04:00
draft: false
---

While writing the mount resource for [mgmt](https://github.com/purpleidea/mgmt)
I needed to restart the `local-fs.target` and `remote-fs.target` systemd units
in order to mount any new fstab entries. To restart a unit using `godbus/dbus`
you can do the following:

```
// conn is a *dbus.Conn connection
sd1 := conn.Object(
    "org.freedesktop.systemd1",
    dbus.ObjectPath("/org/freedesktop/systemd1/"),
)
if call := sd1.Call(
    "org.freedesktop.systemd1.Manager.RestartUnit",
    0,
    "local-fs.target",
    "fail",
); call.Err != nil {
    return errors.Wrapf(call.Err, "error restarting unit: %s", unit)
}
```

When you employ a dbus call to change the state of a systemd unit, it will return
right away, likely before the state has changed. If you want to wait until the
job is complete, which you probably should, you can wait until the `JobRemoved`
signal corresponding to the call sends a message, and process the result to
be sure the state was changed successfully. The following example function restarts
the given dbus unit and waits for it to finish starting up. If context timeout
is exceeded, it will return an error.

```
func restartUnit(conn *dbus.Conn, unit string) error {
    // Timeout context should be longer than the unit takes to start up.
    ctx, cancel := context.WithTimeout(context.TODO(), 30*time.Second)
    defer cancel()

    // Add a dbus rule to watch the systemd1 JobRemoved signal.
    args := fmt.Sprintf(
        "type='signal', path='%s', interface='%s', member='%s', arg2='%s'",
        "/org/freedesktop/systemd1",
        "org.freedesktop.systemd1.Manager",
        "JobRemoved",
        unit,
    )
    if call := conn.BusObject().Call(
        "org.freedesktop.DBus.AddMatch",
        0,
        args,
    ); call.Err != nil {
        return errors.Wrapf(call.Err, "error creating dbus call")
    })
    // Remove the match when we're done, and ignore the error since we're
    // shutting down.
    defer conn.BusObject().Call("org.freedesktop.DBus.RemoveMatch" 0, args)

    // channel for dbus signal
    ch := make(chan *dbus.Signal)
    defer close(ch)

    conn.Signal(ch)
    defer conn.RemoveSignal(ch)

    // restart the unit
    sd1 := conn.Object(
        "org.freedesktop.systemd1",
        dbus.ObjectPath("/org/freedesktop/systemd1/"),
    )
    if call := sd1.Call(
        "org.freedesktop.systemd1.Manager.RestartUnit",
        0,
        unit,
        "fail",
    ); call.Err != nil {
        return errors.Wrapf(call.Err, "error restarting unit: %s", unit)
    }

    // wait for the job to be removed, indicating completion
    select {
        case event, ok := <-ch:
        if !ok {
            return fmt.Errorf("channel closed unexpectedly")
        }
        if event.Body[3] != "done" {
            return fmt.Errorf("unexpected job status: %s", event.Body[3])
        }
    case <-ctx.Done():
        return fmt.Errorf("restarting %s failed due to context timeout", unit)
    }
    return nil
}

```

Hope this is helpful for anyone who needs to wait for systemd to do its thing.
Hit me up with any questions in the comments below, or ping me on twitter
[@JonWritesCode](https://twitter.com/JonWritesCode)
