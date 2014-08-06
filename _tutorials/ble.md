---
layout: tutorial
title: Bluetooth Low Energy
description: How does Bluetooth Low Energy work?
image: http://lorempixel.com/g/400/400/
order: 10
---

This tutorial will provide a brief introduction to the basics of Bleutooth Low Energy.

## GAP

GAP _(Generic Access Profile)_, controls connections and advertising in Bluetooth.
GAP is what makes your device visible to the outside world, and determines how two devices can interact with each other.

GAP defines various roles for devices, but the two key concepts to keep in mind are Central devices and Peripheral devices.
Peripheral devices are small, low power, resource contrained devices that can connect to a much more powerful central device.
The Pucks we're going to be creating are typical peripheral devices.
The smart phone app which is communicating with all the pucks is a typical central device.

## Advertising

The peripheral devices broadcasts advertising data, which lets central devices discover them, and see what kind of services they implement, what their device name is, etc.

## Services

A bluetooth service is a set of features that a device implements.
One device may implement several of these services.
For instance, a health thermometer would implement a 'Health thermometer' service.
A complete list of BLE services can be found on the [Bluetooth Developer Portal](https://developer.bluetooth.org/gatt/services/Pages/ServicesHome.aspx).
When no standard BLE service exist for your use, you are free to define your own service with a custom UUID.

## Characteristics

Each service has a set of characteristics assosicated with it.
A characteristic is a value that can be written and read over Bluetooth.
For the Health Thermometer service, the temperature measurement will be one characteristic.
