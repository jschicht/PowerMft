PowerMft is a truly powerful tool for low level modifications to the system files on NTFS volumes.

Regard it as highly experimental, because it is more of a hack tool for NTFS than anything else.

The tool will let you modify MFT records directly on disk. The following parts of a given record are currently supported:
The record header.
$STANDARD_INFORMATION attribute.
$FILE_NAME attribute.
$I30 index in $INDEX_ROOT attribute (parent of target).
$I30 index in $INDEX_ALLOCATION attribute (parent of target).
$O index in $ObjId at $INDEX_ROOT/$INDEX_ALLOCATION (relevant for files containing $OBJECT_ID attribute).
$R index in $Reparse at $INDEX_ROOT/$INDEX_ALLOCATION (relevant for files containing $REPARSE_POINT attribute).
$DATA in $AttrDef (the Attribute Definition Table).

Given that the $O and $R index already are supported in $INDEX_ROOT/$INDEX_ALLOCATION it would be trivial to implement support for modifying $OBJECT_ID and $REPARSE_POINT attributes.

These are the parameters the tool can accept:
Variables for the MFT record header:
	/HdrSignature is the record signature, usually FILE. (4 bytes)
	/HdrUSAOffset is the offset to the usa. (2 bytes)
	/HdrUSASize is the size of the usa. (2 bytes)
	/HdrUSANumber is the replacement value at sector end. (2 bytes)
	/HdrUSA is the usa values. (HdrUSASize x2 bytes)
	/HdrLsn is the $LogFile sequence number. (8 bytes)
	/HdrSequenceNo is the sequence number of this MFT record. (2 bytes)
	/HdrHardLinkCount is the hardlink count. (2 bytes)
	/HdrAttributeOffset is the offset to the first attribute. (2 bytes)
	/HdrFlags is the flags that indicate file/folder and active/deleted. (2 bytes)
	/HdrRecordRealSize is the real size of this record. (4 bytes)
	/HdrRecordAllocatedSize is the allocated size of this record. (4 bytes)
	/HdrBaseRecord is the base record for this MFT record. (6 bytes)
	/HdrBaseRecordSequenceNo is the sequence number of the base record. (2 bytes)
	/HdrNextAttributeId is the id of the next attribute. (2 bytes)
	/HdrPadding is a 2 byte padding. (2 bytes)
	/HdrMftRecordNumber is the MFT record number. (8 bytes)

Variables for the $STANDARD_INFORMATION attribute:
	/SICTime is the timestamp File Create Time. (8 bytes)
	/SIATime is the timestamp File Modified Time. (8 bytes)
	/SIMTime is the timestamp MFT Entry modified Time. (8 bytes)
	/SIRTime is the timestamp File Last Access Time. (8 bytes)
	/SIFilePermission is the timestamp. (4 bytes)
	/SIMaxVersions is the Maximum number of Version. (4 bytes)
	/SIVersionNumber is the Maximum allowed Versions for a file. (4 bytes)
	/SIClassID is the Class Id. (4 bytes)
	/SIOwnerID is the Owner Id. (4 bytes)
	/SISecurityID is the Security Id key into the index of $SII and $SDS. (4 bytes)
	/SIQuotaCharged is the number of bytes this file accupy for user quota if quota enabled. 0 means disabled. (8 bytes)
	/SIUSN is the Update Sequence Number in $UsnJrnl. (8 bytes)

Variables for the $FILE_NAME attribute:
	/FNParentReferenceNo is the MFT number of the parent. (6 bytes)
	/FNParentSequenceNo is the sequence number of the MFT of parent. (2 bytes)
	/FNCTime is the timestamp File Create Time. (8 bytes)
	/FNATime is the timestamp File Modified Time. (8 bytes)
	/FNMTime is the timestamp MFT Entry modified Time. (8 bytes)
	/FNRTime is the timestamp File Last Access Time. (8 bytes)
	/FNAllocSize is the Allocated size on disk. (8 bytes)
	/FNRealSize is the Real size on disk. (8 bytes)
	/FNFlags is the file flags and attributes. (4 bytes)
	/FNUnknownEaReparse is the timestamp (4 bytes)
	/FNNameLength is the number of characters in file name. (1 bytes)
	/FNNameSpace is the filename namespace. (1 bytes)
	/FNFilename is the file name. (FNNameLength bytes)

Variables for $AttrDef:
	/ADVariable can be some combination of variables from the Attribute Definition Table in $AttrDef:
	/ADExistingAttrName is for Attribute Name in attribute definitions in $AttrDef. (128 bytes including padding)
	/ADAttrName is for Attribute Name in attribute definitions in $AttrDef. (128 bytes including padding)
	/ADAttrCode is for Attribute Code in attribute definitions in $AttrDef. (4 bytes)
	/ADDisplayRule is for Display Rule in attribute definitions in $AttrDef. (4 bytes)
	/ADCollationRule is for Collation Rule in attribute definitions in $AttrDef. (4 bytes)
	/ADFlags is for Flags in attribute definitions in $AttrDef. See explanation. (4 bytes)
	/ADMinLength is for Attribute Minimum length/size in attribute definitions in $AttrDef. (8 bytes)
	/ADMaxLength is for Attribute Maximum length/size in attribute definitions in $AttrDef. (8 bytes)

Although the tool supports a large amount of fields to be modified, it does not mean it will work after the modification. You may for instance set invalid value that the system automatically detects and fixes (usually deletes the file).

Bear in mind this is an experimental tool and there is a high chance you mess up the target volume and loose data on it. I take no responsibility for any damage done with this tool. Consider it as for educational purposes. That said, from the limited testing done so far, it seems any corruption is limited to the target file/folder only and at worst chkdsk will remove the corrupted file/folder.

This opens up for some interesting experiments if you can arbitrarily set any of the above mentioned fields in a record. This is essentially what SetMace does, although SetMace only touches timestamps. However the code was adapted from SetMace, but it does not modify shadow copies.

Limitations
All attributes and field sizes are fixed. That means for instance it is currently not possible to adjust filename length. A filename length adjustment would consequently need adjustment to the size of $FILE_NAME attribute, and also size adjustments and recalculations of the $I30 index in $INDEX_ROOT/$INDEX_ALLOCATION of parent (which again would require adjustments to a number of places in the parent's $MFT record). Other size adjustment fields like HdrUSAOffset, HdrRecordRealSize, HdrAttributeOffset and FNNameLength also makes no sense in modifying as the remaining work for it to succeed is not yet in place.  

Warning
Setting values incorrectly will most likely cause Windows to either, fix the incorrect value, delete the file or folder (with content), or in worst case being unable to fix the volume (unlikely though). Be prepared for dataloss on target volume when using this tool. Even though the tool have been through basic testing, there are many possible scenarios the tool have not been tested in. Writing to the physical disk directly, is inherently very risky. The advantage is that you effectively bypass most restrictions otherwise imposed by the OS. But there is one very important security measure introduced with nt6.x (Vista and later), were and explicit block to direct disk writing was added. There are some

With nt6.x (Vista and later) Microsoft blocked direct write access to within volume space (like \\.\PhysicalDrive0 or \\.\E:): http://msdn.microsoft.com/en-us/library/windows/hardware/ff551353(v=vs.85).aspx
In order to do so it was necessary to dismount the volume first, effectively releasing all open handles to the volume. However this was of course not possible to do on certain volumes (for instance on the systemdrive or a volume where a pagefile existed). The solution to make this work the proper way is to implement a driver that can set the SL_FORCE_DIRECT_WRITE flag on the Irp before sending it further: http://msdn.microsoft.com/en-us/library/windows/hardware/ff549427(v=vs.85).aspx That way, there is no need to dismount the volume, and thus even the systemdrive can be modified. All this, did not apply to nt5.x (XP and Server 2003) and earlier. With 64-bit Windows, Microsoft implemented another security measure, "PatchGuard", that will protect the kernel in memory, and prevent loading unsigned drivers or drivers signed with a test certificate. Of course Windows does not natively ship with a driver allowing to circumvent the security feature mentioned above. Because of this, there are some restrictions to where and when this tool will work.
All secondary volumes, not being systemdrive or drive with pagefile on, are supported.
All volumes are supported for nt5.x (XP and 2003).
On 64-bit Windows (Vista and later), systemdrive or drive with pagefile, will need a driver loaded for the modification. The OS will have to be booted with TESTSIGNING configured.
On Vista and later (32-bit), the systemdrive or drive with pagefile on, will need a driver loaded for the modificaation. As the OS don't have PatchGuard, the driver can be loaded arbitrarily for administrators.

Note that for 64-bit OS the boot configuration of TESTSIGNING is only drivers being signed with selfsigned certificates. If the certificate is signed with a proper certificate, you will have to spend some on this. Here is a list of your options; https://msdn.microsoft.com/en-us/library/windows/hardware/dn170454(v=vs.85).aspx The hacky way is to crack the Windows kernel.

Explanation of some fields

/FNFlags or /SIFilePermission are the dwFlagsAndAttributes parameter of CreateFile is a bitmask of:
read_only = 0x0001
hidden = 0x0002
system = 0x0004
archive = 0x0020
device = 0x0040
normal = 0x0080
temporary = 0x0100
sparse_file = 0x0200
reparse_point = 0x0400
compressed = 0x0800
offline = 0x1000
not_indexed = 0x2000
encrypted = 0x4000
directory = 0x10000000
index_view = 0x20000000

/FNNameSpace is the filename namespace:
POSIX = 0
WIN32 = 1
DOS = 2
WIN32 & DOS = 3

/FNFileName is the filename as defined in $FILE_NAME attribute. Depending on the filename length and hard links there may be several $FILE_NAME attributes for a given MFT record. For longer filenames, there will usually be 2 $FILE_NAME attributes present, 1 for DOS and 1 for WIN32. Although in recent Windows versions it is possible to deactive 8.3 naming style (DOS style for shorter names) for certain volumes. See NtfsDisable8dot3NameCreation in registry. For hard links there will also be created a $FILE_NAME attribute for every hard link a file has got. This means there can actually be a rather large amount of $FILE_NAME attributes for a given file. When modifying this attribute, it is important to understand that specifying target with MFT ref you must set /FNForceFileName. The reason for this is that there may be cases when you want to specify target by MFT ref. In these cases the filename will be set in those $FILE_NAME attributes with same filename length. Specifically in situations with invalid filenames, this may be handy. Other times you may want to (test) set multiple duplicate filenames (for instance in hard links). Then you are able to set the new filename on all $FILE_NAME attributes with the same filename length. The /FNFileName can also be set with hex. For specification with hex, pre-fix with 0x, so for instance a filename of test.txt becomes /FNFileName:0x74006500730074002E00740078007400. This specification of filename in hex is extremely powerful. One challenge might be when /Target is an invalid file name, and Windows is unable to unable to identify the file part out of this; E:\folder\invalid/|\filename.ext. For this there is a parameter /FNCoreFileName where you would set /FNCoreFileName:invalid/|\filename.ext. Alternatively in such case one could identify the MFT ref and specify that in target. For instance, for MFT ref 456, you would set /Target:C:456 and /FNForceFileName:1 (For this my RawDir tool would work great to get the MFT ref first). When filenames are modified there is a slight chance that the $I30 index in the parent is incorrectly sorted. This is a minor error that chkdsk will fix (the index needs to be alphabetically sorted by filename).

Reparse point work differently though as they have their own MFT record, but as the $REPARSE_POINT attribute and the corresponding entry in the $R index in $Reparse points to some other object on the filesystem.

Can Windows fix invalid filenames itself?
The short answer is no in most cases. What chkdsk will do is simply delete the file, which is a horrible fix for a semi-trivial task.

/ADFlags is some flags for an entry in the Attribute Definition Table in $AttrDef. Presumed meaning:
ZERO = 0x0000 (unknown)
INDEXABLE = 0x0002 (This flag is set if the attribute may be indexed)
DUPLICATES_ALLOWED = 0x0004 (This flag is set if the attribute may occur more than once, such as is allowed for the File Name attribute)
MAY_NOT_BE_NULL = 0x0008 (This flag is set if the value of the attribute may not be entirely null, i.e., all binary 0's)
MUST_BE_INDEXED = 0x0010 (This attribute must be indexed, and no two attributes may exist with the same value in the same file record segment)
MUST_BE_NAMED = 0x0020 (This attribute must be named, and no two attributes may exist with the same name in the same file record segment)
MUST_BE_RESIDENT = 0x0040 (This attribute must be in the Resident Form)
LOG_NONRESIDENT = 0x0080 (Modifications to this attribute should be logged even if the attribute is nonresident)

Examples
PowerMft.exe /Target:c:\bootmgr /Verbose:1
(Will just dump the MFT record for the bootmgr file on volume c)

PowerMft.exe /Target:D:\testfile.txt /HdrLsn:999999
(Set the $LogFile sequence number to 999999 in the record header for the D:\testfile.txt)

PowerMft.exe /Target:D:\testfile.txt /Verbose:1 /HdrLsn:999999
(Set the $LogFile sequence number to 999999 in the record header for the D:\testfile.txt. Also dump to console both original and modified MFT record)

PowerMft.exe /Target:D:\testfile.txt /SISecurityID:0
(Set SecurityID to 0 in $STANDARD_INFORMATION for the D:\testfile.txt)

PowerMft.exe /Target:D:198 /SIQuotaCharged:256 /SIUSN:1000
(Set QuotaCharged to 256 bytes and USN to 1000 in $STANDARD_INFORMATION for file with MFT record number 198 on the D volume)

PowerMft.exe /Target:D:\folder\file.ext /SIOwnerID:45 /FNParentSequenceNo:3000
(Set OwnerID to 45 in $STANDARD_INFORMATION and ParentSequenceNo to 3000 in $FILE_NAME for the file D:\folder\file.ext)

PowerMft.exe /Target:D:\folder /HdrSequenceNo:20 /FNMTime:"2000:01:01:00:00:00:789:1234"
(Set Sequence number to 20 in record header and LastWriteTime to 2000:01:01:00:00:00:789:1234 in $FILE_NAME for the folder D:\folder)

PowerMft.exe /Target:D:\folder\test.txt /FNFileName:dumb.txt
(Set the filename of D:\folder\test.txt to D:\folder\dumb.txt in $FILE_NAME and the $I30 index)

PowerMft.exe /Target:D:\folder\te|/.txt /FNCoreFileName:te|/.txt /FNFileName:te__.txt
(Change the invalid filename of D:\folder\tes|/txt to D:\folder\te__.txt in $FILE_NAME and the $I30 index)

PowerMft.exe /Target:D:\file.ext /FNFileName:0x20002000200020002000200020002000
(Change the filename of D:\file.ext to the invisible file "D:\        " with a filename consisting of 8 spaces in $FILE_NAME and the $I30 index. Just for fun.)

PowerMft.exe /Target:"D:\        " /FNCoreFileName:"        " /FNFileName:file.ext
(Rename back the invisible file with the 8 spaces to D:\file.ext in $FILE_NAME and the $I30 index.)

PowerMft.exe /Target:D:4 /ADExistingAttrName:$REPARSE_POINT /ADAttrName:$CHKDSK_UNHAPPY
(Access the Attribute Definition Table in $AttrDef and change the name of $REPARSE_POINT to $CHKDSK_UNHAPPY)

PowerMft.exe /Target:D:4 /ADAttrName:$CHKDSK_UNHAPPY /ADAttrCode:272 /ADDisplayRule:0 /ADCollationRule:0 /ADFlags:128 /ADMinLength:0 /ADMaxLength:16384
(Access the Attribute Definition Table in $AttrDef and create the new attribute $CHKDSK_UNHAPPY)